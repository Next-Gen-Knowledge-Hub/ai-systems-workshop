# Large language models, prompting, and agents

Under every agent is a probabilistic token machine. Training teaches it to predict the next token. At inference it runs the context through learned attention and feedforward weights and emits a distribution. Sampling picks one token, and the loop continues until an end marker. What feels like intelligence is that fast prediction. Alignment, including learning from human feedback, teaches the model to follow instructions and stay helpful. Fresh facts and stronger reasoning come from other training, and from the prompt you write.

Tokens are pieces of text, often smaller than words, and braces and field names get counted too. The same facts in JSON can cost far more than prose. Providers bill output tokens at a higher rate than input, and a long prompt makes the next token less certain. Measure with the model's tokenizer. Temperature flattens or sharpens the distribution. Nucleus sampling drops the low-probability tail. A max-token cap stops rambling and bounds the bill. I usually move only temperature and that cap, and I tune temperature or top-p while leaving the other at its default. Temperature zero narrows the pick. Explicit instructions do the real stabilizing. A seed helps when a debug run must repeat. Keep reasoning effort low on routine steps, and spend it on planning and stuck loops.

The persona is how you make the agent predictable. Name the role, put the task first, fence the user's data with one delimiter style, state numeric bounds, show a few labeled examples, ask for silent step-by-step thought, and say what to do. Write it the way you would brief a sharp new hire who shares none of your assumptions. Keep stable instructions at the top so a prompt cache can reuse them, and put the changing content at the bottom.

Prompts fail in familiar ways: too many topics in one window, rules that contradict, a tiny prompt that causes a round trip per branch, or so many constraints that some get ignored. When a prompt will not sit still, split it into role-sized steps. An agent keeps looping until the goal is met or clearly out of reach, so describe the decisions and the branches.

The OpenAI Agents SDK is the light runtime I use to turn that prompt into a running agent. Pin the model, set the knobs for the role, and type the output so the next step receives a schema. Tracing shows every model call and tool call, in order, with latency. Tools create the agency. Keep the list short. Chain a call when one result feeds the next, and issue independent calls together.

- The model proposes a distribution. Your sampling strategy chooses the token.
- A coder wants a low temperature so syntax can repeat. A writer wants more spread and a nudge toward new ideas.
- Typed outputs plus traces keep a stochastic planner debuggable once tools enter the loop.
- Read the trace before you rewrite the prompt. The order of the tool calls is the plan the model actually ran.
