# 2. LLMs, prompting, and agents

Companion notes for **Chapter 2** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

This chapter is the first time you build. The model is a probability machine.
The persona is how you bias that machine. The SDK is the loop that turns a
prompt into something you can pin, type, trace, and arm with tools. Skip this
chapter and you will treat GPT as a person, treat prompts as vibes, and ship
an "agent" that is a `print()` around a chat completion. Then temperature,
JSON, and a second tool will each surprise you as if they were unrelated bugs.

## The mental model

Everything later in the Agents track sits on this pipeline. If you cannot
point to which box a failure came from, you will retune the prompt forever.

```
  text (system + user + tool results)
       |
       v
  [ TOKENIZE ]     bytes -> integer ids in a vocabulary
       |
       v
  [ FORWARD ]      same ids in  =>  same logits out
                   (the "brain" is a next-token distribution)
       |
       v
  [ SAMPLE ]       temperature, top-p, top-k, penalties, max tokens
                   pick ONE id from the distribution
       |
       v
  [ DETOKENIZE ]   id -> text fragment; append; repeat
       |
       +-- until stop / max tokens / a tool-call schema fires
       |
       v
  [ RUNTIME ]      OpenAI Agents SDK: Agent + Runner
                   persona (instructions)  |  typed I/O  |  traces
                   tools: your functions now; MCP servers later
```

The one sentence to remember a year from now: the model proposes the next
token; you own everything around the proposal — sampling, the persona,
schemas, the tool runtime, and the trace.

Two consequences fall straight out of that diagram. First, "the model
decided" is almost never a complete diagnosis: sampling, the prompt, the
schema, or the tool observation may have decided. Second, a minimal `Agent`
with only `instructions` is still a prompt step. Agency starts when the
runtime can choose a tool and come back. That gap is why this chapter ends
on tools.

## Understanding LLMs

You do not need a transformer derivation to ship agents. You do need a
picture that makes token bills, "why did it say that?", and temperature
debates stop being folklore.

### Probabilistic token machines

Teams talk about LLMs as if they know, intend, or retrieve. Then a planner
invents a source, a JSON field flips type, and someone files it as a
"hallucination bug in the API." Name the machine you actually run.

An LLM is trained to score **what token is likely to come next**, given the
tokens so far. Training chewed through text (and, later, preference data)
until those scores got good enough to be useful. At inference time the
story is smaller:

1. Your prompt is tokenized into ids.
2. The network emits a **logit** for every id in the vocabulary.
3. A **decoding / sampling** step turns logits into one id.
4. That id is appended. Repeat.

The forward pass is **deterministic for a given input and weights**. The
same prompt, same model revision, same nothing-random in the net: same
distribution. Variation you see from run to run is almost always the
sampler, plus anything you changed in the prompt (including hidden system
text, tool schemas, and prior tool results).

That split is the whole job of this section. Creativity is how you carve a
distribution. Factual tightness is how sharply you peak the distribution,
plus whether the right facts are even in the context. Greedy decoding
(always take the argmax) is a sampler with no randomness. Nucleus / top-p,
top-k, and temperature are other samplers. Agents use all of them depending
on the role of the step: a planner that must emit five stable tasks wants
greedy-ish settings; a brainstorming specialist may want more mass in the
tail.

Because the model only ever emits a token, structure is a trick you play on
the sampler. You play it with prompts, with JSON schemas and tool calls, and
with typed `output_type` later in this chapter. You do not play it by hoping.

A second operational fact: the model does not execute your Python. When you
"give it a tool," you give it a **schema**. It emits a structured call. Your
runtime runs the function and stuffs the observation back into the next
tokenize step. That is still this same machine. Tools do not grant knowledge;
they grant grounded next tokens.

### What is a token?

A token is an integer in a model-specific vocabulary: a word, a word piece, a
space-plus-word, a punctuation cluster, sometimes a whole common phrase.
Tokenization is the map from bytes to those ids. You do not get to pick the
map. The provider did, when they trained.

People budget and debug in characters or words. Bills, context windows, and
"why did this JSON blow the window?" are in **tokens**. Measure tokens.
Never infer them from text length.

Consequences you will hit in the first week:

- **English prose is cheap-ish.** Common words often land as one token.
  Rare names, code identifiers, and other-language text fragment more.
- **JSON is expensive for the same facts.** Braces, quotes, commas,
  repeated keys (`"description":`) are tokens too. A compact sentence and a
  pretty-printed object with the same payload can differ by 2× or worse. Use
  JSON when a schema needs it (tool args, typed outputs). Do not narrate a
  novel in JSON "for structure."
- **Whitespace and fences cost.** Markdown fences, indent, and duplicated
  system reminders are not free.
- **Providers price input and output separately.** Output is usually dearer.
  A chatty persona is a cost centre. A planner that emits five short tasks is
  a different product from a planner that writes essays.
- **The context window is a token budget, not a character budget.** Stuffing
  the transcript, the full tool list, and a PDF dump is how you hit the wall
  in week two. Session truncation and retrieval manage that budget later; the
  unit they manage is the token you met here.

Practical habit: count with the tokenizer that matches the model (the
tiktoken family for many OpenAI-compatible models, or the SDK's usage fields
on the trace). If your eval set is "about 800 words," convert it. If two
prompts "look the same length" and one is JSON, they are not the same
experiment.

Tool schemas sit in the context too. Ten chatty tools with long docstrings
can cost more than the user message. That is a preview of the last section:
the tool list is part of the prompt. When something "doesn't fit," ask: ids
in, ids reserved for the completion, ids burned on tools and history. A short
question can still exhaust the window.

### Temperature, top-p, and related knobs

Sampling knobs change **how you pick** from the distribution. They are not
the same as HTTP retries, API keys, or timeouts. Those control the call.
These control the token. Mixing the two in a config dump is how on-call
cannot tell a 429 from a wild completion.

```
  logits
    |
    |  temperature  -- sharpens or flattens the distribution
    v
  probabilities
    |
    |  top-k / top-p  -- optionally zero-out the tail
    |  presence / frequency penalty -- push against repetition
    v
  draw one token     (or argmax if temperature is 0 / greedy)
    |
    |  max_tokens / stop sequences -- hard stop, not a style hint
    v
  append and repeat
```

**Temperature.** Low (including 0) peaks on the mode: more repeatable plans,
tighter JSON, less "creative" phrasing. High flattens: more diversity, more
chance of a fluent wrong fact. For agent steps that must be evaluated, start
low. You can always add a specialist later whose job is variation. You cannot
eval a planner that emits a different graph every run and then blame the
tools.

**Top-p (nucleus).** Keep the smallest set of tokens whose cumulative
probability exceeds *p*, then sample inside that set. Low top-p is another
way to cut the tail. Pick one primary knob per agent, document it, and leave
the other at a sane default unless you are running a real experiment. Two
knobs moving in one deploy is one data point with no attribution: the eval
moved, and you cannot say which knob helped.

**Top-k.** Hard cap on how many tokens remain after sorting. Less common in
current OpenAI-style agent defaults; you will meet it on some local runtimes.
Same idea: truncate the tail before drawing.

**Max tokens.** A **cap**, not a target. The model does not "use up" the
budget for quality. It stops when it hits the cap, a stop sequence, or a
natural end. Too low: truncated JSON, half a tool call, a plan that ends at
step 3 of 5. Too high: a runaway narrator and a bill. Set it from the schema
of the step (five short tasks vs a long report), not from a global constant
copied between agents.

**Frequency and presence penalties.** Nudge against repeating tokens already
in the completion (frequency) or against tokens that appeared at all
(presence). Useful for some creative jobs. For structured agent output they
are usually noise: you want the same key names every time.

**Stop sequences.** Extra brakes. Handy when a legacy prompt still speaks in
prose. Prefer typed outputs (below) over stop-sequence archaeology.

**Reasoning / effort knobs on "thinking" models.** Frontier vendors now
expose an extra axis: how much internal deliberation to spend before the
visible tokens. Defaults are often medium or high. That default is usually
wrong for an agent. An agent is many small, well-scoped steps. The persona
and the tools already constrain the work. High effort on every hop adds
latency and hidden tokens without changing the act. Set effort **minimal /
none** on routine tool hops; raise it on the few steps that are actually hard
(a nasty plan, a synthesis). This is the same discipline as temperature: per
role, per hop.

One global `temperature=0.7` on a multi-step research graph is a classic
incident: the planner drifts, the extractor invents fields, the writer is
accidentally fine. Pin **model + sampling + max tokens + reasoning effort**
on each `Agent`. Treat them as part of the persona's contract. If you need a
creative sibling, it is a second agent with its own settings, a second mood
in the same constructor.

## Prompt engineering as the persona layer

Layer 1 is **persona**: role, expertise, tone, operating constraints. Prompt
engineering is the craft of writing that layer so the token machine is boring
in the right ways.

Models got better; the craft did not vanish. What vanished is the need for
incantations ("you are GPT-4, take a deep breath"). What remains is clear
work: who you are, what "done" looks like, what you must not do, where the
inputs sit, which tools exist. A frontier model with a muddled persona still
muddles. A smaller model with a sharp persona often wins the eval.

Facts about your company do not belong here. They belong in knowledge and
retrieval. Mixing them is how a system prompt becomes a wiki and a compliance
nightmare. Style and policy that must hold even when retrieval misses:
persona. Tickets, SKUs, last week's incident: knowledge.

### Core techniques

These patterns keep showing up because they match how the models were
trained: role-play, format imitation, local coherence, instruction following.
They are not vendor-specific magic. Use them until a typed schema or a tool
makes a given trick unnecessary.

**Assign a role that sets vocabulary.** "You are a senior incident commander"
does more than flattery. It pulls the distribution toward severity,
timelines, and comms, away from blog-post warmth. For agents, the role also
names which tools are in character. A researcher may search; a clerk may
file; a "helpful friend" will do both and also apologize.

**Say the task, then the constraints, in that order.** Front-load what to do.
Put length, tone, and refusal rules next. Models overweight the start and the
end of a prompt; they lose the middle of a manifesto. If a constraint
matters, it is a bullet near the task, a footnote after three paragraphs of
philosophy.

**Make the output shape explicit — unless the SDK already owns it.** "Five
numbered tasks, five words or fewer" is a prompt-level schema. Once you set
`output_type` (below), stop restating the JSON keys in prose. Duplicated
schemas drift. Pick one source of truth.

**Delimit regions.** User text, policies, examples, and untrusted tool output
should not look like one soup. Markdown headings, XML-ish tags, or fenced
blocks give the sampler edges. The point is not aesthetics. The point is so
you can point at a region when a pitfall hits (contradiction, injection,
example that looks like a new instruction).

**Few-shot when the format is fiddly.** One or two *short* examples of the
I/O you want beat a paragraph of negatives. Bad few-shots become the
distribution: if every example is witty, the agent will be witty on a refund.
For typed outputs, prefer a schema plus one example object, a short story.

**Positive instructions over a wall of "don't."** "Write five tasks" beats
"Don't write essays, don't use commas, don't number from zero." Negatives are
leaks: they name the failure mode and sometimes summon it. Use a "don't" when
the model keeps doing a specific wrong thing you have seen in traces.

**Tool-use instructions are part of the persona.** "First call
`get_research_sources`, then plan only from that list" is not a nice extra.
Without it, a capable model will skip the tool and plan from parametric
memory. You will only see that if you trace.

**Chain-of-thought as a prompt trick vs as a layer.** Asking the model to
think step by step can help a single completion. Deep patterns (ReAct, trees,
Reflexion) are a reasoning layer of their own. Do not paste a full ReAct
spec into every persona "for quality." You will pay tokens and still need the
runtime loop.

A compact map for an agent persona:

```
  ROLE          who, and whose tools
  TASK          what "done" is
  PROCESS       order of tools / checks (short)
  CONSTRAINTS   length, safety, what not to invent
  I/O           schema or "the SDK type owns this"
  EXAMPLES      only if the type is still ambiguous
```

If a block cannot be named with one of those labels, it is probably layer 4
(facts) or layer 3 (a planning protocol) in the wrong house.

### Thinking like an LLM

The useful analogy is not "talk to a genius." It is brief a competent new
hire who has no memory of your company and no eyes except the tokens you
pass. They have read the public internet of the training cutoff. They have
not read your Jira. They will fill gaps with something that sounds like your
industry.

Write the task as you want it executed. "When you research X, always consider
Y and Z" is closer to what will happen than "be thorough." You can describe a
small workflow in the persona: fetch sources, filter to allowed ones, emit
five tasks. The model will often follow that flowchart — until a tool returns
something ugly. Then you need the real loop (tools, observations, a
termination condition), a longer flowchart in prose.

A prompt that assumes shared context — "use the usual sources," "follow our
style," "you know what I mean by urgent" — fails silently. Put the definition
in tokens. Names of sources. A one-line style rule. A severity rubric. If it
must change weekly, it is not a persona constant; retrieve it.

Specificity is not the same as length. "Time-travel story, 1921, five
paragraphs, humorous first person" is specific and short. A two-page essay on
"what good research means" is long and still vague. Prefer testable clauses:
counts, lists, "only from the tool result," "if the tool errors, say so."

Untrusted text (web pages, tickets, PDFs, other agents) will appear inside
the same window. Thinking like an LLM means assuming that text can look like
instructions. Delimit it. Tell the persona that content inside a `USER` or
`TOOL` region is data. The full threat model for deployment is larger; the
habit starts when you write the first delimiter.

### Common pitfalls

Prompt work fails in a handful of boring ways. Treat them as defects in
layer 1, not as model mystery.

**Too complicated.** One persona tries to research, write, cite, refuse
medical advice, joke, and speak two brands. The window fills, instructions
collide, cost goes up, and the model satisfies a random subset. Split by
role. That may be two `Agent`s and a handoff, or one agent with a narrower
job. "Do one kind of work, then stop" is a feature.

**Contradictory instructions.** "Be concise" next to "explain like a
textbook." "Never mention competitors" next to "compare the market." The
sampler must pick. You will not like the pick. Read the prompt as the model
sees it: one blob, top to bottom, including defaults the SDK injects. Resolve
clashes with an explicit priority bullet. Delete the loser.

**Too simple.** "Help the user" plus twelve tools. The agent either chatters
(many tiny calls, latency, bill) or guesses without calling anything. Either
enrich the process section ("call X before Y") or cut tools until the
remaining set matches the persona. Agency is not "maximum verbs."

**Ambiguous delimiters.** Mixing `"""`, markdown fences, and XML tags until
user input can close a fence and write a new system line. Pick one delimiter
style. Show an example of user text that contains backticks and make sure
your wrapper still holds. This is the cheap cousin of injection defense.

**Unspecified output.** Free prose into the next agent. Comma vs list vs
JSON. The downstream prompt "usually" parses it until Thursday. Typed
outputs (next section). If you cannot type it yet, specify a grammar a human
could check in five seconds.

**Examples that fight the rules.** Few-shots that ignore the length cap, or
that call a tool the production agent does not have. Examples are prompts.
Keep them legal. Delete them when the schema is enough.

**Persona as a junk drawer.** Changelog, API keys, "temporary" debug lines,
copied Slack policy. If it is a secret, it is not a prompt. If it is a fact
that ages, it is memory or retrieval. If it is a debug flag, it is a trace
attribute.

## Building with the OpenAI Agents SDK

Plenty of frameworks exist. This track standardizes on the **OpenAI Agents
SDK** because it is small: an `Agent` is instructions plus model settings
plus tools; a `Runner` executes the loop; traces are first-class. Protocols
(MCP, later A2A) are how you stay portable around that choice. You can
mentally translate every listing here into "persona + sampler + tool runtime
+ trace" on another stack.

You will not learn the SDK by memorizing class names. You will learn it by
noticing what each constructor argument **owns**.

### A minimal agent

A first agent in this book-shaped workshop is usually a **research planner**:
topic in, a short plan out. That is deliberate. It forces you to write a
persona, run a completion, and look at tokens before you drown in tools.

The moving parts:

```
  load credentials (.env) -- not in the prompt, not in git
       |
       v
  instructions = """ persona: role, task, constraints """
       |
       v
  agent = Agent(name=..., instructions=...)
       |
       v
  result = Runner.run_sync(agent, input=user_text)
       |
       v
  result.final_output     # still a string unless you type it
```

This looks like an agent in a demo, so it ships as one. Label it honestly.
With no tools and no loop beyond one completion (or a hidden inner sample
loop that only emits text), you have a **prompt step**. Prompt chaining —
several such steps in code you wrote — is a valid workflow. It is not yet
agency. Agency is the model choosing a registered tool and the runtime coming
back. Keep the planner; add tools before you name the product "agent."

What to get right even at this stage:

- **Name** the agent after the role, not after the company. Traces and later
  multi-agent graphs will show this string.
- **Instructions are the persona.** Put TASK where a tired reader sees it.
  Use delimiters. Specify "five concise tasks" if that is the contract.
- **`input` is the user goal**, not a second system prompt. If you find
  yourself stuffing policy into `input`, it belongs in `instructions` or in
  a tool.
- **Runners** exist in sync and async forms. Sync is fine for scripts and
  tests. Async shows up the moment you host MCP over SSE or wait on multiple
  tools.

Environment setup and sample repos are the book's appendix territory. Do not
paste secrets into instructions to "make the demo work."

### Model and parameters

If you omit the model, you get **whatever the SDK / account currently
defaults to**. That default will move. Your eval set will not. Pin the model
id.

Attach sampling on the agent, not "somewhere in the HTTP client":

- Planner / extractor / anything you snapshot in tests: `temperature` at 0
  (or as close as the vendor lets you), modest `max_tokens` matched to the
  schema.
- Writer / idea generator: higher temperature, still a max.
- Reasoning-effort: **none or minimal** on short hops; raise only on the
  step that needs deliberation.

One shared `ModelSettings` object mutated by every agent in the process is
how traces lie. Settings are part of the agent's identity. Copy explicitly if
you must share a baseline. When an eval fails, you want to read the trace and
see *this* agent's model, temperature, and max tokens, not a global that
changed in another file.

Changing model family is a **release**. Tokenizers differ. Tool-call formats
differ. "It got cheaper" is not the same as "it still emits five tasks." Pin,
measure, then swap — the same rule that applies to prompts and retrieval.

A note on "we will route in production." Routing is real. Here you learn to
**declare** what a role needs: model family, sampling, max tokens, effort.
The platform that chooses a provider from that declaration is a different
job. This chapter owns the declaration.

### Controlling inputs and typed outputs

LLMs emit tokens. Downstream code wants objects. If you glue those worlds
with split-on-comma, you will spend the rest of the workshop debugging the
glue.

Agent A's prose is agent B's input. Variability that was charming in a demo
("here are some ideas!") becomes a parse error, a missed field, or a silently
truncated list. Declare a schema (Pydantic models, TypedDicts, whatever the
SDK accepts as `output_type`). Make the runner **fail the completion** if the
object does not validate. Then pass the object, not a string, to the next
hop.

```
  without types:
    planner --> "1. foo\n2. bar..." --> regex / hope --> next agent

  with types:
    planner --> { tasks: [ {step, text}, ... ] } --> next agent
                ^
                validated, same keys every time
```

What belongs in the type:

- Fields the **next agent or the UI** will read. If nobody reads it, do not
  ask the model to invent it.
- Types that match reality (`HttpUrl`, enums of allowed sources), not `str`
  for everything. Constraining the sampler is the point.
- Docstrings / field descriptions — models see them. Write them like prompt
  fragments: short, operational.

What does not belong:

- Restating the entire persona. The schema is the I/O contract, not the role.
- Optional fields "just in case." Optionality is where the model skips work.

Typed **inputs** matter when a previous hop or a tool returns structure. Do
not stringify a list of sources and hope the planner re-parses it. Pass the
list. The fewer times a fact becomes prose and back, the fewer times it
mutates.

When validation fails, that is a **trace event**, not a retry storm with the
same prompt. Typical causes: `max_tokens` too low, persona still asking for
markdown, temperature too high, model too small for the schema. Fix the
cause. Blind retries train you to ignore a broken contract.

Multi-agent graphs assume this habit. If you skip types here, you will
"debug handoffs" for a week when you are actually debugging strings.

### Tracing

If you cannot see the prompt the model *actually* received — system, tools,
user, prior tool payloads — you are guessing. The OpenAI Agents SDK turns
tracing on by default when you talk to the OpenAI API. The vendor dashboard
is enough to learn the shape: one **trace** for the run, **spans** for LLM
calls and later for tools.

Look at, every time, until it is boring:

- Which **model** and **settings** fired (not which you *meant* to pin).
- **Token counts** in and out. This is your first cost and window signal.
- The **instructions** blob. Confirm the persona you edited is the one that
  ran (no stale deploy, no wrong agent name).
- The **output**. For typed agents, the parsed object *and* any repair/retry
  the SDK did.
- Later: **tool calls** (name, args, observation, latency, error).

Tracing only in the vendor UI, only on OpenAI, only when someone remembers to
click, will not survive a second provider. Learn the dashboard in this chapter
so the *idea* is in your hands. For production, multi-provider, or regulated
shops, plan an external trace store. Here you only need: wrap the run in a
named `trace(...)`, read it, change one thing.

Name traces after the scenario (`"chapter2-planner-tools"`), not `"test"`.
Six months from now you will grep these names. Disable or export carefully in
CI: traces can contain user text and tool payloads. That is a data-handling
choice, not a dashboard preference.

## Tool integration

Tools are how layer 2 gets into code. A tool is a function plus a **schema**
the model can see: name, description, typed arguments. Actions are what
happen when the runtime executes a chosen call.

Without tools you can chain prompts. With tools the model can **branch**:
fetch, then decide. That is the smallest agency this track cares about.

### Providing agents with tools

In this chapter, tools are **in-process functions** registered on the `Agent`
(`tools=[...]`, often via a decorator that builds the schema from the
signature and docstring). Later, the same idea is a server another host can
load. Same contract, different process boundary.

```
  def get_research_sources() -> list[str]:
      """Return the allow-listed research sources for this agent."""
      ...

  Agent(..., tools=[get_research_sources], instructions="""...
        Call get_research_sources before you plan.
        Only plan against that list.
        ...""")
```

The docstring **is** the prompt for that tool. Write it for the model: when
to call, what the args mean, what you will get back. Cute one-liners that
please a linter are how the model calls the wrong verb.

Design rules that save pain:

- **One job per tool.** `get_research_sources` and `get_resource_url` beat
  `do_research(magic: str)`. Small tools compose; god-tools cannot be
  guarded.
- **Keep the list short.** Every schema burns tokens on every turn and
  competes for attention. A confused model with fifteen tools is under-
  specified.
- **Persona must mention the tools that matter.** Registration is necessary,
  not sufficient. If the process says "look up the URL per source," the model
  has a reason to call the second tool.
- **Own failure.** Timeouts, empty lists, 429s, unexpected shapes: return a
  structured error the model can read, and tell the persona what to do
  (replan, tell the user, stop). A demo that only shows happy JSON is still
  chapter 1's warning.
- **Do not put secrets in tool descriptions.** They will be traced and they
  will leak into prompts.

Handoffs to other agents are also "tools" in some graphs. Platform registries,
credentials, and circuit breakers sit outside this chapter. Here you own the
function, the schema, and the observation.

A planner that must only use allow-listed sources is the teaching example for
a reason. If the tool says Wikipedia / Google / YouTube and the plan cites
arXiv, learn failed: either the tool was not called, the observation was
ignored, or the persona did not constrain the act. Tracing tells you which.

### Tracing tool use

Once two tools exist and one depends on the other, traces stop being optional.

"It didn't use the URL tool" is reported as a model quality issue. Sometimes
it is a prompt issue. Sometimes the first tool's observation was too huge.
Sometimes the schema misnamed an argument. Sometimes `max_tokens` ate the
tool-call JSON. Read the span. You want a story like:

```
  LLM 1  -->  tool get_research_sources()
         <--  ["Wikipedia", "Google", "YouTube"]
  LLM 2  -->  tool get_resource_url(name="Wikipedia")
         <--  "https://..."
  LLM 3  -->  (optionally more URLs)
  LLM n  -->  final typed plan that only names those sources
```

If LLM 1 already emits the plan, the persona's "begin by using the tool" is
dead letter. If the tool is called with `name="wiki"`, the description or
enum is weak. If the observation is an exception traceback dumped as a
string, you handed the model a new prompt injection surface.

Wrap the `Runner` in an explicit `trace("...")` when you are comparing runs.
Change **one** thing: instructions, tool docstring, temperature, or schema.
One change per experiment is how you learn which lever moved.

Tool chaining (A's output is B's argument) is powerful and is how agents
become graphs inside a *single* persona. It is also how latency multiplies.
If you need a guaranteed sequence, you can still say so in the persona; do
not assume the model will invent the sequence from two unrelated docstrings.

Eval of "did it call the right tool?" belongs in a later evaluation chapter.
Your job now is to **see** the call. You cannot score a ghost.

## Check yourself

1. A teammate says the planner is "non-deterministic because LLMs are
   random." Which box in this chapter's pipeline is actually drawing the
   token, and what would you set (and pin) before you agreed the *model* was
   the problem? Give a real job (planner vs copywriter) where you would want
   leftover randomness.
2. You paste a 2 KB JSON blob and a 2 KB paragraph of the same facts into a
   tokenizer. Why can the JSON cost more, and what does that do to a
   tool-heavy agent that re-sends schemas every turn?
3. Temperature is 0.2 and top-p is 0.1 on the same agent, changed in one
   deploy, and the eval moved. Why can you not say which knob helped? What
   experiment would you run instead?
4. A "thinking" model's default reasoning effort is high. Your agent does
   eight tool hops to file a ticket. What fails if you leave the default,
   and which hops might still deserve more effort?
5. Rewrite this persona defect: "Be concise but thorough, never guess, and
   use any tools you need to delight the user." Name the pitfall class
   (complicated / contradictory / too simple / …) and a replacement that a
   trace could falsify.
6. Why is a minimal `Agent` with only `instructions` not yet an agent in the
   sense of sense-plan-act-learn? What extra loop would have to show up in a
   trace before you would use the word in a design review?
7. You add `output_type=ResearchPlan` but the instructions still say "reply
   as markdown." A downstream agent sometimes gets markdown anyway. What two
   places do you inspect first, and what does a validation error mean that a
   retry will not fix?
8. The vendor trace UI shows a completion but you are mid-migration to a
   second model provider. What does this chapter still give you about the
   *idea* of a trace, and what do you need to plan for production?
9. `get_research_sources` is registered; the plan cites a source not in the
   list. Walk the failure as persona vs schema vs observation vs sampling.
   What would the trace have to show for each?
10. You want "the org default model, unless this agent needs vision." Which
    part of that sentence is declaring what a role needs (this chapter), and
    which part is choosing a provider at runtime? Why does collapsing them
    make every team reimplement routing?
