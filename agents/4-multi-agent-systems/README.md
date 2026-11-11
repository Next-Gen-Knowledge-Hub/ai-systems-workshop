# Architecting multi-agent systems

The useful question is which shape of agents fits the goal. Early unconstrained loops spawned subgoals without a stop and looked brilliant until they fell over. Later systems answered with roles, channels, and a deliberate architecture. A lone agent still wins when one persona and a short tool list can hold the job. It loses when focus must switch, tools collide, or the window cannot carry the whole problem.

Three axes come before the diagram. Decision-making is who may commit to the next action, and with how much context. Control is who executes. Communication is which tokens may cross the boundary. Too little context and the agent decides blind. Too much context and every call pays for noise, and the important detail disappears in the pile.

Flow is an assembly line: easy to test, and the run stalls if one stage stalls. Orchestration keeps command in a hub, hands control to workers, and can skip or recover. Collaboration lets peers share decisions and talk. It can crack ambiguous work, and it is chatty, slow, and hard to score. A manager over submanagers stretches the hub into a hierarchy. Start with a flow. Add a hub when centralized delegation is actually required.

Match coordination to the dependency, and begin with one strategy. Sequential when each stage needs the previous output. Parallel when the pieces are independent and a later step only merges them. Hierarchical when a boss plans different work for each worker. Iterative critique is one authority revising one worker. Debate is peers arguing toward a consensus. Conditional routing, best-of-N, and role-play are further options once the simple shape is solid.

Send only the context that agent has earned: the assignment, the rubric, the schema. A shared thread keeps history and grows expensive. A typed pass-off gives you a checkpoint and more code. A tool or MCP call can carry a structured result. SDK handoffs move control through the prompt and a handoff list, which is fast and leaves the dependency implicit. Guardrails on input and output reject, retry, or correct before a weak plan contaminates the next agent. Use a deterministic check on any branch that must pass or fail the same way every time.

- Agency is earned. Begin with the least autonomy that solves the problem, and tighten tools, schemas, budgets, and human gates where an action is hard to undo.
- Pick one communication style per codebase. Mixing a shared thread, messages, MCP, and a separate agent-to-agent protocol multiplies the debugging.
- Keep each tool list small and scoped to the role. Every registered tool is paid for on every call.
- A flow I trust has typed inputs and outputs, a picture of the graph, traces, and guardrails you can catch and retry. Choose the pattern that meets the need at the lowest token, latency, and coordination cost.
