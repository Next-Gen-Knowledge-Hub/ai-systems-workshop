# Agent reasoning and planning

An LLM predicts the next token and then the job is gone. A plan survives the next call only when you leave it in the prompt, a tool result, or a store you own. Decomposition asks what the pieces of the goal are. Planning asks the order, the tools, and how those pieces join. A sensible split followed by the wrong sequence fails differently from a tidy sequence built on the wrong pieces, and the repair follows the failure.

Everyday tasks run on habit. The model has neither habit nor a private picture of the room, so those steps have to be written down. A plain model answers in one forward pass and will skip the hard middle step. A reasoning model spends extra test-time compute before the visible answer. Treat that phase as deliberation inside a single call. A typed plan, a tool loop, and an audit trail are still yours to build and to keep.

Add structure when the horizon is long, several tools compete, a mistake is costly, or someone later must see why a step ran. A single reversible lookup can stay a plain completion.

- **Chain-of-thought.** Ask for the intermediate steps in the completion, then the answer. Spend it on dates, units, and local logic. There is no observation in that pass, so a missing fact gets narrated. Show the user the answer. Keep the chain in the log.
- **ReAct.** Reason, act, observe, and let the observation rewrite the next thought. The runner calls the model, executes the tool, appends the result, and calls the model again, under a turn cap. A chain that ends in one tool call is still a single plan. When the job has a join, store the plan and let ReAct execute only the current step.
- **LLM planning.** That stored outline holds subtasks, dependencies, and a check for done, and later hops reread it. Speak in tool names you actually registered. When the plan changes, patch the object so the next hop reads the revision.
- **Tree-of-thought.** Generate candidates, score them, expand the winners, and stop at a budget. Pruning is the point, and the score lives in your code. A sentence that says to consider three options is a list inside one completion. Execute the winner with ReAct, and leave the discarded branches out of the user answer.
- **Reflexion.** Attempt, critique, and attempt again with that critique in the next prompt. The weights stay put, so the gain is a richer next prompt, and only when you can tell a good attempt from a bad one. A hint that worked can be saved and reused on the next problem of the same shape. Keep the retry off any action that must happen only once.

Choose the pattern from the shape of the hop, and keep that choice stable from run to run. Lookups stay a forward pass. Logic with no tools gets a chain. Live data gets ReAct. Plans you can score get a tree. A detectable, reversible failure gets Reflexion. Listing every pattern in one persona lets the sampler pick a different mood on each run.

The sequential-thinking MCP server is the pad those patterns write to. One tool records a thought, a revision, or a branch, and later steps read it back. The model still does the thinking. Prove the agent can see the tool before you pile on a puzzle. A fresh local process starts empty, so a plan that must outlive the process needs a transport that keeps the pad. World tools still own the facts.
