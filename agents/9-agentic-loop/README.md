# Understanding the agentic loop

A loop earns its place when one call cannot finish the job: the facts arrive in stages, each pass improves the draft, a queue of items is waiting, the next move depends on the last result, or the useful sources only appear while you work. Classification, a short answer, and a format change stay one-shot. Extra cycles there only add delay and spend.

Every layer stands on the same four pieces: a goal, a plan, state that survives the cycle, and a clear rule for when to stop.

- **Inner SPAL (sense-plan-act-learn).** The agent reads its state, chooses a next step, uses a tool, checks the result, and carries what it learned forward. It picks the tool and the stop. You supply the goal, the tools, and the boundaries.
- **Task loop.** Keep the long plan and the accumulated state in code outside the model. A general-purpose model is a weak home for a long horizon, and the window fills. Deep research is the usual shape: typed state and plan, tools, and one iteration that returns findings, follow-up questions, a satisfaction flag, and a confidence score, then a termination gate.
- Stack that gate. A hard iteration cap and a cost ceiling are the safety net. Goal-met and a quality bar are the happy exits. Stagnation catches the run that looks busy while back-to-back findings barely change. Leave any of the defensive stops off and the loop is an incident.
- Let the research agent search. After the loop, a separate synthesis agent writes the report from the full trail and marks an early stop. A repetitive task loop is the same idea on a known queue: take one item, record the outcome, retry a transient failure, and stop when the queue is empty. The list already is the plan.
- **Meta loop.** An agent owns the decision, the state, and the plan, so the campaign can change while it runs. Code no longer holds those three.
- Orchestration is one delegator that treats specialist workers as tools and can add or reorder subtasks when the first split was incomplete. Use it when the breakdown itself is uncertain. Each decision is another model call, so a known checklist should stay a plain task loop.
- Collaboration is peers on one shared state. A researcher, a critic, and a synthesizer take turns until they agree the goal is met or the round limit hits. A thin turn manager only chooses whose turn it is.
- Orchestrate independent work such as batch jobs and parallel search. Collaborate when the pieces depend on each other and the output gets better if someone argues with it. If you are unsure, start with orchestration. It is easier to debug. Move to collaboration when quality needs that argument.
