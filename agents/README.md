# AI Agents in Action, Second Edition

**Micheal Lanham** (Manning, 2026). Subtitle: *Intelligent workflows with LLMs, MCP, A2A, and more.*

This book is how I build agents that finish real work, not chatbots that stop after one reply. An agent senses a goal, plans, acts with tools, and learns from what comes back. I organize that capability as five layers — persona, tools, reasoning, knowledge and memory, and evaluation — then show how those layers become multi-agent flows, retrieval, a measured loop, a deployment you can leave running, and a cognitive system that notices when it is stuck.

You need basic Python and comfort with a GitHub repo. No prior agent or prompt-engineering experience. The second edition is organized around those five layers, with MCP, Docker, test-driven evaluation, the agentic loop, and metacognition added on top of the first edition's patterns.

## Contents

1. [The rise of AI agents](./1-rise-of-ai-agents/README.md)
2. [Large language models, prompting, and agents](./2-llms-prompting-agents/README.md)
3. [Actions with the Model Context Protocol](./3-mcp/README.md)
4. [Architecting multi-agent systems](./4-multi-agent-systems/README.md)
5. [Agent reasoning and planning](./5-reasoning-and-planning/README.md)
6. [Working with memory and knowledge](./6-memory-and-rag/README.md)
7. [Building robust agents with evaluation and feedback](./7-evaluation-and-feedback/README.md)
8. [Deploying agents and agentic systems](./8-deploying-agents/README.md)
9. [Understanding the agentic loop](./9-agentic-loop/README.md)
10. [The cognitive agent that thinks, monitors, and adapts](./10-cognitive-agents/README.md)
11. [Tips for building agentic systems](./11-field-tips/README.md)
- [Appendix A: Setting up the sample code](./appendix-a-sample-code/README.md)
- [Appendix B: Node.js setup for local MCP servers](./appendix-b-nodejs-mcp/README.md)

## Takeaways

- Agency is a loop with a stop, not a longer prompt. You still own execution, retries, and memory.
- Give the model a persona, a short tool list, and a typed output. Read the trace before you rewrite the prompt.
- MCP is the shared socket for tools. Trust, credentials, and whether a call should happen stay yours.
- Start with one agent and a flow. Add a hub or a team only when one role cannot hold the job.
- Match the reasoning pattern to the hop: a lookup, a chain, ReAct, a stored plan, a tree, or a Reflexion retry.
- Retrieval is how knowledge and memory enter a finite prompt. Search by meaning and by exact tokens, then compress.
- Robustness is a loop you start on day one: tests, a grounding check, a rubric, and traces you can annotate.
- Ship the smallest runtime that meets the latency, behind budgets, idempotent tools, and a threat model you can draw.
- A cognitive agent picks a strategy, withholds a claim when confidence is low, and changes approach when progress stalls.
