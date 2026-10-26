# 2. LLMs, prompting, and agents

Companion notes for **Chapter 2** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is the machine under the agent. It establishes how a completion is produced, how a persona biases that machine, and how a small SDK pins the model, types the output, traces the run, and arms the model with tools. Afterward you can name the box a strange completion came from: the tokenizer, the forward pass, the sampler, the persona, the schema, or the tool observation. A `print()` around a chat completion stops looking like an agent.

## 1. The proposal pipeline

The model proposes the next token. You own everything around the proposal: sampling, the persona, schemas, the tool runtime, and the trace.

```
  text (system + user + tool results)
       |
       v
  [ TOKENIZE ]     bytes -> integer ids in a vocabulary
       |
       v
  [ FORWARD ]      same ids in  =>  same logits out
       |
       v
  [ SAMPLE ]       temperature, top-p, top-k, penalties, max tokens
       |
       v
  [ DETOKENIZE ]   id -> text fragment; append; repeat
       |
       +-- until stop / max tokens / a tool-call schema fires
       |
       v
  [ RUNTIME ]      Agent + Runner
                   instructions | typed I/O | traces | tools
```

* **Tokenize**: The prompt becomes integer ids. The vocabulary is model-specific. The provider chose it at training time.
* **Forward**: The network emits a **logit** (a raw score) for every id in the vocabulary. Same ids and same weights produce the same distribution. This step is deterministic.
* **Sample**: Temperature, top-p, top-k, penalties, and the token cap pick one id. Run-to-run variation lives here, plus anything that changed in the prompt: hidden system text, tool schemas, prior tool results.
* **Detokenize and repeat**: The id becomes a text fragment, is appended, and the loop continues until a stop, the token cap, or a tool-call schema fires.
* **Runtime**: An `Agent` plus a `Runner` holds the persona, the types, the trace, and the tools. The model still does not run your Python. It names a call. The runtime executes it and puts the observation into the next tokenize step.

"The model decided" is an incomplete diagnosis until you know which box moved. A minimal `Agent` with only `instructions` is still a prompt step. Agency starts when the runtime can choose a registered tool, run it, and come back.

## 2. Next-token machines

An LLM is trained to score which token is likely to come next, given the tokens so far. Training chewed through text, then through preference data, until those scores were useful. Inference is the short version of that story.

* **The four beats**: Tokenize the prompt. Emit a logit per vocabulary id. Sample one id. Append it and repeat.
* **Greedy decoding**: Always take the argmax. A sampler with no randomness. Stable plans and tighter JSON live near this end.
* **Creativity vs tightness**: Creativity is how you spread probability mass into the tail. Factual tightness is how sharply you peak the distribution, plus whether the right facts are in the context at all. The model fills a gap with something that sounds like the domain.
* **Structure is a trick on the sampler**: Prompts, JSON schemas, tool-call formats, and a typed `output_type` change which tokens can win. Hoping for a stable shape does not.
* **A tool grants grounded next tokens**: You show a schema (name, description, typed arguments). The model emits a structured call. Your runtime runs the function and returns an observation. The next forward pass sees that observation as ordinary tokens. The tool does not install new knowledge inside the weights.
* **Same words, different input**: Hidden system text, a tool schema, and a prior tool result are part of the ids. The user sentence can look identical while the forward pass is a different call.
* **The misfile**: A planner invents a source, or a JSON field flips type, and the ticket says "hallucination bug in the API." The weights scored a plausible continuation. The fact was missing from the context, or the sampler left enough tail to emit it.

A planner that must emit five stable tasks wants a peaked sampler. A brainstorming specialist may want more mass in the tail. Those are different roles, so they get different sampling contracts.

## 3. Tokens and the window

A **token** is an integer in that vocabulary: a word, a word piece, a space-plus-word, a punctuation cluster, sometimes a common phrase. People budget in characters or words. Bills, context windows, and truncated JSON are in tokens.

* **Measure, do not infer**: Count with the tokenizer that matches the model (the tiktoken family for many OpenAI-compatible models) or with the SDK's usage fields on the trace. An eval set of "about 800 words" is not a token count until you convert it. Two prompts that look the same length are different experiments when one of them is JSON.
* **English prose is cheap-ish**: Common words often land as one token. Rare names, code identifiers, and text in other languages fragment into more ids.
* **JSON is expensive for the same facts**: Braces, quotes, commas, and repeated keys (`"description":`) are tokens. A compact sentence and a pretty-printed object with the same payload can differ by 2× or worse. Use JSON when a schema needs it (tool arguments, typed outputs). A novel narrated as JSON pays for punctuation.
* **Whitespace and fences cost**: Markdown fences, indentation, and a system reminder copied into every region are tokens.
* **Output is usually dearer**: Providers price input and output separately. A chatty persona is a cost centre. Five short tasks and a five-paragraph essay are different products.
* **The window is a token budget**: The transcript, the tool list, and a pasted PDF all spend the same budget. When a short question still exhausts the window, count ids in, ids reserved for the completion, and ids burned on tools and history.
* **Schemas sit in the context**: Ten tools with long docstrings can cost more than the user message. The tool list is part of the prompt on every turn.

## 4. Sampling knobs

Sampling knobs change how you pick from the distribution. HTTP retries, API keys, and timeouts change the call. Mixing the two in one config dump is how an on-call confuses an HTTP 429 with a wild completion.

```
  logits
    |  temperature                 sharpen or flatten
    v
  probabilities
    |  top-k / top-p               drop the tail
    |  presence / frequency penalty
    v
  one token (argmax if temperature is 0)
    |  max_tokens / stop sequences
    v
  append and repeat
```

Pick one primary knob per agent, write it down, and leave the other at a sane default unless the run is a real experiment. Pin **model + sampling + max tokens + reasoning effort** on each `Agent`. A creative sibling is a second agent with its own settings.

1. **Temperature**: Low, including 0, peaks on the most likely token: repeatable plans, tighter JSON, less decorative phrasing. High flattens the distribution: more diversity, and more chance of a fluent wrong fact. Steps you will evaluate start low. One global `temperature=0.7` on a research graph is a known incident: the planner drifts, the extractor invents fields, and the writer happens to look fine.

2. **Top-p (nucleus)**: Keep the smallest set of tokens whose cumulative probability exceeds *p*, then sample inside that set. A low *p* cuts the tail. Temperature 0.2 and top-p 0.1 changed in the same deploy leave you with one data point and no attribution when the eval moves.

3. **Top-k**: A hard cap on how many tokens remain after sorting by score. Less common in current OpenAI-style agent defaults. Local runtimes still expose it. Same job as top-p: truncate the tail, then draw.

4. **Max tokens**: A cap. The model stops at the cap, at a stop sequence, or at a natural end. It does not spend the budget to raise quality. Too low and you get truncated JSON, half a tool call, or a plan that dies at step 3 of 5. Too high and you get a narrator and a bill. Set the cap from the schema of that step (five short tasks, or a long report). A constant copied from another agent is how the extractor inherits the writer's budget.

5. **Frequency and presence penalties**: Frequency pushes down tokens already repeated in the completion. Presence pushes down tokens that appeared at all. Some creative jobs want that. Structured agent output usually wants the same key names every time, so the penalties are noise.

6. **Stop sequences**: Extra brakes for a legacy prompt that still speaks in prose. A typed `output_type` replaces stop-sequence archaeology once the SDK can reject a bad object.

7. **Reasoning effort**: An extra axis on thinking models: how much internal deliberation to spend before the visible tokens. Defaults are often medium or high. An agent is many small, scoped steps, and the persona plus the tools already constrain the work. High effort on every hop adds latency and hidden tokens. Set effort to **minimal / none** on routine tool hops. Raise it on the few steps that are actually hard: a nasty plan, a synthesis.

### Watch for

* Two sampling knobs moved in one deploy. Change one, rerun the eval, then touch the next.
* A high default reasoning effort left on eight ticket-filing hops. The extra deliberation does not change the act. Raise it only on the hop that is hard.
* `max_tokens` set once for the whole process. The step with the tight schema is the one that gets cut off.

## 5. The persona

**Persona** is the role, the expertise, the tone, and the operating constraints. Prompt engineering is how you write that layer so the token machine is boring in the ways you need.

* **What still earns its tokens**: Incantations ("you are GPT-4, take a deep breath") stopped paying rent. What remains is who you are, what done looks like, what you must not do, where the inputs sit, and which tools exist. A frontier model with a muddled persona still muddles. A smaller model with a sharp persona often wins the eval.
* **Facts live next door**: Style and policy that must hold even when retrieval misses belong in the persona. Tickets, SKUs, and last week's incident belong in knowledge you retrieve. A system prompt that is also a wiki is a compliance problem and an audit problem.
* **A new hire with no eyes**: Brief a competent hire who has read the public internet up to the training cutoff and has not read your tracker. They fill gaps with something that sounds like the industry. "When you research X, always consider Y and Z" is closer to the completion you will get than "be thorough."
* **Shared context fails quietly**: "The usual sources," "our style," and "you know what urgent means" are not in the tokens. Put names of sources, a one-line style rule, and a severity rubric in the prompt. A definition that changes every week is retrieved. It is a poor persona constant.
* **Specific and short**: "Time-travel story, 1921, five paragraphs, humorous first person" is specific. A two-page essay on "what good research means" is long and still vague. Prefer clauses a trace can falsify: counts, lists, "only from the tool result," "if the tool errors, say so."
* **Untrusted text can look like an instruction**: Web pages, tickets, PDFs, and other agents share the window. Delimit them. Tell the persona that a `USER` region and a `TOOL` region are data.

A short flowchart in the persona — fetch sources, keep the allowed ones, emit five tasks — often holds until a tool returns something ugly. The real loop is tools, observations, and a stop condition. A longer essay in the prompt does not grow that loop.

## 6. Instructions the sampler can follow

These patterns keep showing up because training rewarded role-play, format imitation, local coherence, and instruction following. They are not vendor magic. Drop a trick once a typed schema or a tool makes it unnecessary.

* **Role**: "You are a senior incident commander" pulls the distribution toward severity, timelines, and comms, and away from blog-post warmth. The role also names which tools are in character. A researcher may search. A clerk may file. A "helpful friend" will do both, and apologize.
* **Task, then constraints**: Put the task first. Put length, tone, and refusal rules next to it. The start and the end of a prompt outweigh the middle of a manifesto. A constraint that matters is a bullet beside the task.
* **One output shape**: "Five numbered tasks, five words or fewer" is a prompt-level schema. Once `output_type` is set, stop restating the JSON keys in prose. Two copies of a schema drift. One of them is the source of truth.
* **Delimiters**: User text, policy, examples, and tool output need edges: a heading, an XML-ish tag, or a fence. The point is that you can point at a region when a contradiction, an injection, or a bad example shows up.
* **Few-shot**: One or two short examples beat a paragraph of negatives when the format is fiddly. Examples are part of the distribution. If every example is witty, the refund reply will be witty. For a typed output, prefer the schema plus one example object.
* **Positive instructions**: "Write five tasks" beats "don't write essays, don't use commas, don't number from zero." A negative names the failure and sometimes summons it. Keep a "don't" for a specific wrong move you have already seen in a trace.
* **Tool order**: "Call `get_research_sources`, then plan only from that list" is the contract. A capable model will otherwise skip the tool and plan from parametric memory. The skip is invisible until you read the trace.
* **Step-by-step vs a reasoning method**: "Think step by step" can help a single completion. ReAct, tree-of-thought, and Reflexion need a runtime that actually iterates. Pasting a full method spec into every persona spends tokens and still does not run the loop.

```
  ROLE          who, and whose tools
  TASK          what "done" is
  PROCESS       order of tools and checks (short)
  CONSTRAINTS   length, safety, what not to invent
  I/O           the schema, or "the SDK type owns this"
  EXAMPLES      only while the type is still ambiguous
```

A block that cannot take one of those labels is a fact, or a planning protocol, sitting in the persona by mistake.

## 7. Persona defects

Prompt failures are defects in this layer. They are boring, and they are readable.

1. **Too complicated**: One persona researches, writes, cites, refuses medical advice, jokes, and speaks two brands. Instructions collide, the window fills, and the model satisfies a random subset. Split by role: two agents and a handoff, or one agent with one job. Stopping when that job is done is a feature.

2. **Contradictory**: "Be concise" beside "explain like a textbook." "Never mention competitors" beside "compare the market." The sampler picks a side. Read the blob the model sees, top to bottom, including defaults the SDK injects. Add an explicit priority bullet and delete the loser.

3. **Too simple**: "Help the user" plus twelve tools. The agent either chatters (many tiny calls, latency, a bill) or guesses and calls nothing. Write the process ("call X before Y") or cut tools until the set matches the persona. A long verb list is not agency.

4. **Ambiguous delimiters**: `"""`, markdown fences, and XML tags mixed until user text can close a fence and write a new system line. Pick one style. Feed it user text that contains backticks and confirm the wrapper still holds.

5. **Unspecified output**: Free prose into the next agent. Comma-separated, or a list, or JSON, until Thursday. Type the object. If you cannot type it yet, specify a grammar a person could check in five seconds.

6. **Examples that fight the rules**: A few-shot that ignores the length cap, or that calls a tool the production agent does not have. Examples are prompts. Keep them legal. Delete them when the schema is enough.

7. **Junk drawer**: A changelog, an API key, a temporary debug line, a Slack policy pasted in. A secret is an environment variable. A fact that ages is retrieval. A debug flag is a trace attribute.

### Watch for

* "Be concise but thorough, never guess, and use any tools you need to delight the user" is contradictory and too simple at once. A replacement a trace can falsify names the tool, the count, and the source of facts: "Call `get_research_sources` first. Emit five tasks. Use only names from that list."
* A clash you cannot see in your file can still be in the blob. SDK-injected defaults sit in the same prompt.

## 8. Agent, Runner, and pinned settings

The **OpenAI Agents SDK** stays small on purpose. An `Agent` is instructions, model settings, and tools. A `Runner` executes the loop. Traces are first-class. On another stack the same four pieces are persona, sampler, tool runtime, and trace. A shared protocol such as MCP is how a tool stays portable around that choice. Learn each constructor argument by what it owns.

```
  credentials in the environment     not in the prompt, not in git
       |
       v
  instructions = the persona
       |
       v
  agent = Agent(name=..., instructions=...)
       |
       v
  result = Runner.run_sync(agent, input=user_text)
       |
       v
  result.final_output                a string until you type it
```

* **Name**: Name the agent after the role. Traces show this string. A first build in this shape is a research planner: a topic in, a short plan out. That forces a persona and a look at tokens before tools drown the picture.
* **`instructions` vs `input`**: Instructions are the persona. `input` is the user goal. Policy you find yourself stuffing into `input` belongs in `instructions` or in a tool.
* **An honest label**: No tools, and no loop beyond one completion, means a prompt step. Several such steps in code you wrote is prompt chaining. It is a valid workflow. The word *agent* waits until a trace shows the model choosing a registered tool and the runtime coming back with an observation.
* **Sync and async**: `Runner.run_sync` is enough for scripts and tests. Async shows up when a server is hosted over SSE, or when several tools are in flight.
* **Secrets**: Load them from the environment. Pasting a key into instructions to make a demo run puts the key in the prompt and in the trace.

Each constructor argument owns one job. Memorizing class names teaches less than noticing the owner.

* **`name`**: The string traces show. Use the role.
* **`instructions`**: The persona. Task, constraints, tool order.
* **`model`**: The pinned id. Leave it empty and the account default is free to move.
* **Model settings**: Temperature, max tokens, penalties, reasoning effort. They belong to this agent.
* **`tools`**: The schemas available on this hop.
* **`output_type`**: The object contract. One source of truth, in the constructor.
* **`input` on the runner**: The user goal for this run, not a second persona.

Omit the model and you get whatever the SDK or the account currently defaults to. That default moves. The eval set does not. Pin the model id on the agent.

* **Settings belong to the agent**: A planner, an extractor, or anything a test snapshots: `temperature` at 0, or as close as the vendor allows, and a modest `max_tokens` matched to the schema. A writer: a higher temperature, and still a cap. Reasoning effort stays none or minimal on short hops.
* **Do not share one mutable settings object**: One `ModelSettings` instance that every agent mutates makes traces lie. Settings are part of the agent's identity. Copy a baseline if you need a shared starting point. When an eval fails, the trace should show *this* agent's model, temperature, and max tokens.
* **A model-family change is a release**: Tokenizers differ. Tool-call formats differ. A cheaper model is a different experiment from "it still emits five tasks." Pin, measure, then swap.
* **Declare, then let a platform route**: This chapter owns the declaration: model family, sampling, max tokens, effort. A platform that picks a provider from that declaration is a different job. "The org default, unless this agent needs vision" mixes them. The role's need is the declaration. Choosing the provider at runtime is routing. Collapsing the two makes every team rebuild routing.

## 9. Typed outputs and traces

The model emits tokens. The next hop wants an object. Charming prose ("here are some ideas!") becomes a parse error, a missed field, or a silently truncated list.

```
  without types:
    planner --> "1. foo\n2. bar..." --> regex, then hope --> next agent

  with types:
    planner --> { tasks: [ {step, text}, ... ] } --> next agent
                same keys every time, or the completion fails
```

* **`output_type`**: A schema the SDK accepts, such as a Pydantic model or a TypedDict. The runner fails the completion when the object does not validate. Pass the object onward. `result.final_output` stays a string until you do this.
* **Fields that earn a slot**: Only fields the next agent or the UI will read. Types that match reality (`HttpUrl`, an enum of allowed sources) constrain the sampler. `str` for everything does not. Field descriptions are prompt fragments: short, and operational. The model sees them.
* **Fields that do not**: A restatement of the persona. Optional fields "just in case." Optionality is where the model skips the work.
* **Typed inputs**: A previous hop or a tool that returns a list should pass the list. Stringifying it so the planner can re-parse it is another chance for the fact to mutate.
* **A validation error is a trace event**: The completion failed the schema. Fix the cause. Retrying the same prompt teaches you to ignore a broken contract.
* **Causes you will actually see**: `max_tokens` cut the object off. The persona still says "reply as markdown." Temperature is high enough to wander off the schema. The model is too small for the type. `output_type=ResearchPlan` beside "reply as markdown" is two sources of truth. Inspect the instructions and the type first.
* **Strings become fake routing bugs**: The next hop, given prose, fails in ways that look like a bad handoff. The failure is the untyped string. A typed object removes that class of bug.

If you cannot see the prompt the model actually received — system text, tools, user text, prior tool payloads — you are guessing. The SDK turns tracing on by default for the OpenAI API. The vendor dashboard is enough to learn the shape: one **trace** for the run, **spans** for model calls and for tool calls.

* **Model and settings on the span**: What fired, including a default you did not mean to pin.
* **Token counts**: Input and output. The first cost signal and the first sign the window is filling.
* **Instructions blob**: The persona that ran. A stale deploy and the wrong agent name both show up here.
* **Output**: For a typed agent, the parsed object and any repair the SDK attempted before it failed the schema.
* **Tool spans**: Name, arguments, observation, latency, error.
* **Name the scenario**: Wrap the run in `trace("chapter2-planner-tools")`, not `trace("test")`. You will grep these names. Traces contain user text and tool payloads. Turning them off or exporting them in CI is a data-handling choice.
* **The idea survives a second provider**: A dashboard that exists for one vendor, and only when someone clicks, will not. The fields you keep are the same: the run, the spans, the actual prompt, the actual settings, the actual tool observation. A second provider means an external trace store that holds those fields. Learn the shape here so you know what to put in it.

## 10. Registering a tool

A **tool** is a function plus a schema the model can see: a name, a description, and typed arguments. Prompt chaining can only follow the path you wrote. A tool lets the model branch: fetch, read the observation, then decide. That branch is the smallest agency this chapter cares about.

Here the tools are in-process functions on the `Agent` (`tools=[...]`), often built by a decorator from the signature and the docstring. The same contract can live in another process that a host loads. Either way the model emits a call, and something you run returns the observation.

```
  def get_research_sources() -> list[str]:
      """Return the allow-listed research sources for this agent."""

  Agent(..., tools=[get_research_sources],
        instructions="Call get_research_sources before you plan. "
                     "Only plan against that list.")
```

* **The docstring is the tool's prompt**: When to call, what the arguments mean, what comes back. A one-liner written for a linter is how the model picks the wrong verb.
* **One job**: `get_research_sources` and `get_resource_url` compose. `do_research(magic: str)` cannot be guarded, because you cannot say what it is allowed to touch.
* **A short list**: Every schema is billed on every turn and competes for attention. Fifteen tools and a vague persona is an under-specified model.
* **Mention the tool in the persona**: Registration puts the schema in context. The process section is why the model calls it. "Look up the URL per source" is a reason to call the second tool.
* **Return a structured failure**: Timeouts, empty lists, HTTP 429s, and surprising shapes should come back as text the model can read. The persona says whether to replan, tell the user, or stop. A demo that only returns happy JSON leaves that choice unwritten.
* **Descriptions are public**: A secret in a docstring lands in the prompt and in the trace.
* **A handoff is the same shape**: In some graphs the model names another agent the way it names a function. The runtime performs the transfer. A registry of agents, credential storage, and circuit breakers are platform work. You still own the schema and the observation.

## 11. Reading a tool trace

Two tools, one depending on the other, make the span the only honest record. "It didn't use the URL tool" might be the persona, a huge observation, a misnamed argument, or a token cap that ate the tool-call JSON.

```
  LLM 1  -->  get_research_sources()
         <--  ["Wikipedia", "Google", "YouTube"]
  LLM 2  -->  get_resource_url(name="Wikipedia")
         <--  "https://..."
  LLM n  -->  typed plan that only names those sources
```

* **The plan appears at LLM 1**: "Begin by using the tool" never fired. The persona is a dead letter, or the sampler ignored it. The span shows no tool call.
* **`name="wiki"`**: The description or the argument enum is too loose. The call happened, and the argument is wrong.
* **arXiv in the plan**: The tool returned Wikipedia, Google, and YouTube. Either the tool was not called, the observation was ignored, or the persona did not constrain the act. Match the cited name against the observation in the span.
* **A traceback as the observation**: The model received a new pile of text that can look like instructions. Return a structured error instead.
* **A guaranteed order is written down**: A's output as B's argument is how one persona becomes a small graph, and how latency adds. Two unrelated docstrings will not invent the sequence. Say the order in the persona.
* **One change, then read the span**: You can score "did it call the right tool?" only after the call is visible. A ghost call has no score.

Change one lever and read the next span. Two levers in one run leave you with a difference and no cause.

* **Instructions**: LLM 1 starts calling the tool, or it still emits the plan immediately.
* **Docstring**: The argument stops arriving as `wiki` and starts arriving as a name the tool actually returns.
* **Temperature**: The typed plan stops adding a source that was not in the observation.
* **Schema**: An enum of allowed names makes `wiki` an invalid argument before the sampler has to guess.
* **`max_tokens`**: The tool-call JSON is cut off in the span. Raise the cap to the size of the call, then rerun. A second prompt edit will not repair a truncated call.

## Check yourself

1. A teammate says the planner is non-deterministic because LLMs are random. Which box in the pipeline draws the token, and what do you pin before you agree the model weights are the problem? Name one job where leftover randomness is the point, and one where it is not.
2. You tokenize a 2 KB JSON blob and a 2 KB paragraph of the same facts. Why can the JSON cost more, and what does that do to an agent that re-sends tool schemas every turn?
3. Temperature is 0.2 and top-p is 0.1, both changed in one deploy, and the eval moved. Why is there no attribution? What experiment do you run instead?
4. A thinking model's default reasoning effort is high. The agent takes eight tool hops to file a ticket. What do you pay if you leave the default, and which hops still deserve more effort?
5. This persona is broken: "Be concise but thorough, never guess, and use any tools you need to delight the user." Name the defect classes and write a replacement a trace could falsify.
6. Why is an `Agent` with only `instructions` still a prompt step? What must show up in a trace before you use the word *agent* in a design review?
7. You set `output_type=ResearchPlan`, and the instructions still say "reply as markdown." A downstream hop sometimes receives markdown. Which two places do you inspect first, and what does a validation error mean that a retry of the same prompt will not fix?
8. The vendor trace UI shows a completion, and you are adding a second model provider. Which fields of a trace are the idea you keep, and what do you plan so those fields exist for both providers?
9. `get_research_sources` is registered. The plan cites a source that is not in the returned list. What would the trace show if the cause were the persona, the schema, the observation, or the sampler?
10. Someone says "use the org default model, unless this agent needs vision." Which half of that sentence is the role's declaration, and which half is choosing a provider at runtime? What do you lose by collapsing them?
