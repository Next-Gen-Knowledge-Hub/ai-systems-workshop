# The rise of AI agents

A chatbot answers and stops. An agent perceives a situation, decides, and keeps acting until the goal is done or a limit stops it. Agency is that continuing loop.

I sort the everyday patterns by how much autonomy you hand over. Direct chat only generates text. A tool-augmented model makes one call because you asked for one. An assistant can reach several tools and still waits for approval at each step. An agent receives a goal and chooses the steps. High-stakes actions stay behind a human gate. Autonomy means the agent schedules the work, with oversight designed in.

Inside, the rhythm is sense, plan, act, learn. Take in the goal and the latest observation, break the goal into tool-sized tasks, let your runtime execute the call, then continue, revise, or stop. The shape is old: observe-orient-decide-act, belief-desire-intention, and the sense-plan-act loop from robotics. A language model widens what one agent can reason about. You still own execution, retries, and what gets remembered.

A tool is a schema the model can choose and a function your code actually runs. Feed one result into the next call. Treat timeouts, bad payloads, and rate limits as ordinary control flow, and cap the retries. The Model Context Protocol is the shared socket: an open JSON-RPC standard so an agent can discover tools on a server. Vet those servers, pin versions, and run them with least privilege.

I map capability as five layers that interact the whole time: persona, tools, reasoning, knowledge and memory, and evaluation. The picture organizes the design, and during a run the layers consult each other continuously. Persona is role and constraint. Tools are how the agent touches the world. Native reasoning covers short, cheap, reversible work. Add a structured method when the horizon is long, the action is hard to undo, or you need a trace you can audit. Knowledge is the shared library. Memory is this user and this session. Evaluation scores the step in the loop and the run after it.

One agent stalls when the tool list, the window, or the job outgrows a single role. Split for a reason: specialization, parallelism, a smaller context, or a problem that is several actors. Then pick the cheapest shape that holds. Flow is an assembly line. A hub treats specialists as tools. A peer team can debate, and it spends more. Pass work on a shared thread, on named slots, or as a typed message, and send only what the next worker needs.

- Registering a schema leaves execution, failure, and approval with your runtime.
- Built-in reasoning helps. Clear instructions still steer long or risky work.
- MCP makes a capability portable. Credentials, trust, and whether the call was wise stay yours.
- Start multi-agent work as a flow. Move to a hub or a team when one line of work cannot hold the goal.
