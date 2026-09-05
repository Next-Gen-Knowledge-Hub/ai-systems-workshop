# 9. Building an AI assistant

Companion notes for **Chapter 9** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is the **assembly**. Chapters 2–8 were services. Here
they become one assistant — **Claw** in the book — grown one
capability at a time so you can see which service earns which
behavior. Skip it and you will "know the platform" as a slide and
still paste `openai.chat` into a script when a deadline appears.

The Agents track is a different book. Field patterns for support,
RAG, and research live in
[agents ch. 11](../../agents/11-field-tips/). Mention that folder.
Do not copy its five-layer tips into this file. This folder stays
on **how the DAS platform composes** into a deployed assistant.

## The mental model

```
  Slack / web / API
          |
          v
     API gateway  (auth, trace, route)
          |
          v
  +------ @workflow Claw (small function) ----------------------+
  |  1 input guardrails                                         |
  |  2 session + long-term memories                             |
  |  3 assemble context (budget)                                |
  |  4 loop:  model  --tool calls-->  tools (+ arg policy)      |
  |           ^______________ observations ______________|      |
  |  5 output guardrails                                        |
  |  6 store exchange; stream or return                         |
  +------+-----------+------------+-------------+---------------+
         |           |            |             |
         v           v            v             v
      Model       Session       Data      Tools+Guardrails
         |           |            |             |
         +-----------+-----+------+-------------+
                           v
              Observability + Experimentation
```

The one sentence to remember: **Sarah's code is an orchestrator;
the platform is the product.** If the workflow file is 2,000 lines
of HTTP and prompt glue, you did not use chapters 2–8. You
reimplemented them.

Chapter 1 traced Maria's return-policy question through boxes.
This chapter *is* that trace, with a name and a deploy command.
Persona and context-engineering rows in
[`TRADEOFFS.md`](../../TRADEOFFS.md) are the agent-side cousin;
here context is **assembly from platform sources**.

## The blueprint

Before the first `chat()`, know the target. Claw is a personal
workplace assistant: talk in Slack or a web box, remember the
person, ground in company docs, take actions (calendar, tickets,
web) under policy, show its work in traces, improve through
experiments, deploy as a workflow.

Every box in the diagram is a service you already built. The
application is the **top** function. That inversion is the whole
book: the 2% model call sits at the leaf of a tree that already
exists.

Channels are clients of the **gateway**, not extra copies of
Claw. Slack adapter and the web app POST the same `api_path`.
If each channel reimplements session ids, you are back in sprawl
on day one of "the complete architecture."

Write your own blueprint in one page: who the user is, which
indexes, which tool namespaces, which policies, sync vs stream.
If you cannot name the services, you are drawing a demo.

## The simplest assistant: a model call in a workflow

Every project starts here and should **admit** it. Version 1 of
Claw: `@workflow`, register a system prompt in the Model Service,
`platform.models.chat(...)`, return the text. No memory. No RAG.
No tools.

That is not a toy because the SDK is thin. It is a toy because
**the product is a toy**. It is already a *platform* citizen:
retries, routing, cache, and provider fallback from
[chapter 3](../3-model-service/) apply. You did not write them.
The architecture diagram is one box under the function, and that
is honest.

Failure mode you want to feel: the user says "as I said," and
Claw does not know. Do not "fix" that with a bigger prompt. The
next section exists.

Register the prompt **by name** (`claw-assistant`), not as a
string literal only in the handler. Versioning and A/B later
depend on that registry. A 40-line prompt inlined in Python is
how experiments become git diffs.

## Remember conversations (session)

[Chapter 4](../4-session-service/) is the fix. Load or create a
session, pass history into the model call, append the exchange
afterward. Postgres (or whatever backend you chose) outlives the
pod. Conversations survive deploys and replica hops.

Handler changes are small on purpose: `session_id` in, 
`get_or_create`, `history` into `chat`, `append` user + assistant.
The client must **send the id back** (cookie, Slack thread map,
JSON field). If you key only on `user_id` and cram all history
into one session, you get one infinite transcript and a token
blowup. If you mint a new session every message, you get version
1 again.

Truncation / summary / hierarchical memory from chapter 4 start
to matter as soon as the chat is longer than a demo. Version 2
can still be "send recent turns." Version 2 **must not** be
"stuff until the API errors."

```
  v1:  message --> Model --> text
  v2:  session.get --> Model(history) --> session.append
```

Test: turn 1 "my name is Priya"; turn 2 "what's my name?" If
that fails, stop adding RAG. You do not have a conversation.

## Long-term memory across sessions

A sliding window is this sitting. Tomorrow's session starts
empty unless you store **facts that outlive the transcript**.
Chapter 4's model-managed memories: the model (or a dedicated
pass) proposes what to keep — "prefers window seats", "on-call
for payments this week" — and the Session Service stores them
per user.

Architecture change: the function now receives **two** bundles
from Session: recent messages, and a compact memory list. Both
go into context. Memories are small; their effect on "this
assistant knows me" is large.

Do not dump memories into the Data Service index. "Priya prefers
window seats" is not an HR PDF. Mixing them is how a preference
becomes a retrieval miss and how a PDF becomes fake biography.

Write policy: what may be remembered (preferences, project
nicknames), what must not (passwords, health details your
compliance forbids). Guardrails later can block storing the
latter; Session should not be a silent vault for whatever the
model whispered.

Test: close the session, open a new one, ask "what seat do I
like?" If it fails, you built a chatbot, not an assistant.

## Grounding: knowledge retrieval and agentic RAG

Memories are about the user. **Documents** are about the org.
Without [chapter 5](../5-data-service/), Claw answers HR
questions from pretraining and confidence. That is the dangerous
kind of fluency.

**Naive RAG (always retrieve):** `data.search` / `hybrid_search`
on every turn, stuff top chunks, generate. Good for "what is
the parental leave policy." Cheap to write. Wasteful on "hello"
and on "schedule a meeting with Sam." Wrong when the first query
string is a bad search.

**Agentic RAG:** retrieval is a **tool** (or an explicit step
the model can request). The model may reformulate, search
again, or skip. Latency and loop risk go up; relevance on messy
questions goes up. TRADEOFFS has this row. Agents chapter 6
teaches the agent-shaped version; here you **call Data**, maybe
more than once, from the loop in later sections.

```
  always-on RAG                agentic RAG
  --------------               -------------
  every turn: search           model may search(query')
  k chunks always in context   0..n searches
  simple                       extra hops, extra traces
```

Index choice is a product decision: `hr.policies` is not
`eng.runbooks`. Isolation from chapter 5 is how Claw does not
"helpfully" cite a sealed memo. Filters (`audience=employee`)
belong in the search call, not in a prompt that says "please
only use public docs."

Grounding is not a citation aesthetic. If the chunks do not
support the claim, you need a **score** (chapter 7) and maybe a
critic (Agents ch. 7). This chapter's job is to **put the
chunks in the window** and to let the loop retrieve again.

## Tools and the agent loop

Text that is wrong is embarrassing. A calendar invite to 500
people is an incident. [Chapter 6](../6-tools-and-guardrails/)
is how Claw **acts** without holding vendor keys.

Register tools on the platform (`claw.calendar.list`,
`claw.tickets.create`, `claw.web.search`, maybe
`claw.knowledge.search` if retrieval is a tool). The workflow
discovers a **kit**, not the union of the org. Pass schemas to
the model. When the model returns tool calls, `tools.execute`,
append observations, call the model again. That loop is the
agent.

```
  model --> text? --> done
        --> tool_calls? --> validate args --> execute --> observe --> model
                         (repeat until text or cap)
```

This is ReAct-shaped. The **control** lives in the workflow
function. The Tool Service is the governed socket. Do not
implement OAuth in the handler. Do not `requests.post` to Jira
with a token from env.

Caps: max iterations, max tools per turn, timeout. Uncapped
loops are how a weekend research job becomes a Monday invoice.
Chapter 8's workflow timeout is the outer envelope; the loop
needs an inner counter too.

Side effects: prefer read tools before write tools in the kit
you expose to a general assistant. Confirmation policies (next
section) for anything that emails, spends, or invites.

Agents chapters 4 and 9 go deeper on handoffs and layered
loops. Claw can stay **one** agent with tools. If you split
into a hub of specialists, that is a composition of workflows
or an in-process graph — do not confuse those with extra
platform services.

## Safety: guardrails at every step

Without policy, Claw is a liability with a friendly name.
Joking "invite the whole company," medical advice, a
performance complaint pasted into a public ticket: predictable,
not hypothetical.

Chapter 6's defense in depth maps onto Claw as **inspection
points**:

1. **Input** — jailbreak, PII, disallowed topics, before you
   spend retrieval and model tokens.
2. **Tool arguments** — after the model proposes a call, before
   Execute. Schema is not enough: "this calendar is the
   authenticated user's," "invite list size ≤ N," "ticket
   project in allow list."
3. **Output** — leakage, toxicity, medical disclaimers, before
   the user (or Slack) sees text.
4. **Behavioral** — which tools this application may even
   discover; confirmation for irreversible writes.

Prompt-only "you must not…" is still in the system prompt as
instruction, not as enforcement. Enforcement is the Guardrails
Service and Tool Service policy.

False positives block real work; measure them (chapter 7). A
silent drop is indistinguishable from a hang. Return a clear
refusal **from the policy**, not a model improvisation that
apologizes incorrectly.

If Claw is healthcare-adjacent, input policy is not optional
color. If Claw is "internal only," you still need behavioral
rules; internal users jailbreak for fun.

## Context engineering (the prompt that assembles platform sources)

You have been stuffing things into the window. At some length
the window is a **budget**, not a backpack. No single service
can decide the mix: Session would keep all turns, Data would
keep all chunks, Tools would keep all schemas, the prompt
registry would keep a novella of instructions. **Context
engineering** is the workflow's job: assemble, rank, cut.

Six sources, five services (plus the user):

```
  +------------------ context window ------------------+
  | 1 system prompt     Model registry (versioned)     |
  | 2 long-term memory  Session                        |
  | 3 retrieved docs    Data                           |
  | 4 tool schemas      Tool discovery                 |
  | 5 recent turns      Session (truncated/summarized) |
  | 6 current message   user                           |
  +----------------------------------------------------+
```

Starve history → lose the thread. Flood docs → hallucinated
links between unrelated PDFs. Flood tools → tokens spent
reading JSON the model will not call. Irrelevant memories →
creepy or wrong personalization.

A practical assembly order (teaching, not dogma):

1. System prompt (stable, versioned, includes how to use
   tools and when to search).
2. Memories (small, high value) — hard budget, e.g. N facts.
3. Current user message (must fit; if it does not, reject or
   chunk the *user*, do not silently clip a legal question).
4. Tool schemas for the **kit**, not the planet.
5. Retrieved chunks with a token cap and citations.
6. History into the **remaining** budget (chapter 4
   algorithms).

The system prompt in the book grows by version as capabilities
appear. That is correct: v1 cannot mention tools you have not
wired. Keep capabilities honest. "You can refund" with no
refund tool is how you get hallucinated refunds.

Token accounting should be visible on the trace (chapter 7).
If you cannot say how many tokens were prompt vs tools vs
docs vs history, you cannot tune this.

## The complete agent loop

Put the pieces in one flow. Boxes are platform calls. Diamonds
are decisions that make this an **agent**, not a pipeline.

```
  message in
      |
      v
  input guardrails --block--> refusal
      |
      v
  session + memories
      |
      v
  assemble context (six sources)
      |
      v
  +-- model generate <----------------------------+
  |       |                                       |
  |       +-- text --> output guardrails          |
  |       |              |                        |
  |       |              v                        |
  |       |         append session --> out        |
  |       |                                       |
  |       +-- tool calls --> arg policy --block--+|
  |                         |                    ||
  |                         v                    ||
  |                       execute                 |
  |                         |                     |
  |                         v                     |
  |                      observe -----------------+
  |                       (iteration cap)
```

If input policy blocks, do not search, do not generate. If the
model wants tools, validate each call, execute, feed results
back as messages, loop. If iteration cap hits, return a
controlled failure ("I could not finish this") rather than the
last half-tool thought.

Store **tool observations** in the session if the next user
turn needs them; they are history. Store **memories** only
through the memory path, not by hoping the transcript lasted.

This loop can stay in one workflow function. It can also call
child workflows (chapter 8) for heavy research. Start in one
function until a stage has different scale or GPU needs.

### Streaming

A single JSON at the end is easy to compose and slow to feel.
Layer [chapter 3](../3-model-service/) `chat_stream` with
[chapter 8](../8-workflow-service/) `response_mode="stream"`.

Each loop iteration streams. **Content tokens** yield to the
client immediately. **Tool-call deltas** buffer silently; the
user should not see broken JSON arguments. When the turn ends:
if text, you are done (after output policy); if tools, execute
and iterate.

Output guardrails are the subtle part. Sync Claw can
`filter_output` on the full string. Stream Claw cannot wait
for the whole answer without killing the point of streaming.
A `filter_output_stream` that yields by sentence (or another
safe span), buffering just enough to apply policy, is the
teaching compromise. Some classes of leak **cannot** be
caught until the sentence ends; accept a few hundred
milliseconds of buffer. Do not yield raw tokens past a
policy that exists to stop them.

Errors mid-stream: send a sentinel and close. Partial answers
plus a silent hang are worse than a visible stop.

## Observability

Once Claw is in production, Sarah's questions are operational:

- Why was this reply 8s when most are 3s?
- Dollars per user per day?
- Which searches return junk?
- Is quality rising or are people just tired of filing bugs?

She should **not** add spans by hand around every
`platform.*` call. Chapter 7's default instrumentation already
wraps service boundaries: Model records tokens, cost, TTFT,
model id; Data records query and hit counts; Tools record
duration and status; Guardrails record policy and action.
The waterfall **is** the architecture diagram with timings.

Actionable numbers: a 5s Data span with 0 hits is a grounding
bug, not a "slow LLM." A cheap model with a terrible
helpfulness score is a routing bug. A tool span that fails
then retries into success is an adapter flake.

Custom spans belong on glue you wrote (a homemade rerank, a
rules engine). If the waterfall has a mysterious gap, that
glue is unmarked.

Generations contain user text. ACL on trace viewers is part
of shipping Claw, not a later compliance surprise.

## Experimentation

Observability says what happened. Experimentation says whether
a change **helped**. Example from the book-shaped story: Claw
picks `web.search` when it should have hit the internal index.
Hypothesis: tool guidance is too late in the prompt. Treatment:
reorder sections (new prompt version). Do not ship on taste.

Create an experiment: control = current prompt name/version,
treatment = `claw-assistant-v5-tools-first`, sticky assignment
on `user_id`, metrics: tool-selection accuracy, helpfulness,
cost, latency. Run until you have enough traffic or you hit a
guardrail (quality collapse → abort).

Change **one** thing. If you also bump the model and `k`, you
will not know what to promote.

Online scoring rules should already be sampling Claw. The A/B
just **slices** those scores by variant. Offline dataset first
if the change is dangerous; A/B if the remaining risk is
distributional (real users do not talk like YAML).

Promote by moving the workflow's `system_prompt_name` (or
target pointer) to the winner. Deprecate the loser so the
registry does not become a museum.

## Deploying

The `@workflow` decorator on Claw **is** the deploy contract
from chapter 8: name, `api_path` (`/claw/chat`), mode
(`stream` for the product you actually want), replicas,
CPU/memory, timeout large enough for a few tool hops but not
for infinite research.

`genai-platform deploy claw.py` (or your module) builds the
image, registers, deploys, waits for ready, **registers the
gateway route**. Slack and the web app point at that path.
They do not SSH to pods.

Secrets: still credential refs on tools, not env in the
workflow image. Model provider keys live in the Model
Service. If Claw's Dockerfile contains `OPENAI_API_KEY`,
Sarah skipped chapter 3.

Rollbacks: previous workflow version, not "un-edit the
prompt in prod by hand." Prompt-only changes can also be
registry versions without a new image if inference reads
the name at request time — know which changes need a
rebuild (code, deps) and which need a target promotion
(prompt). Mixing those is how you ship yesterday's code
with tomorrow's prompt and cannot tell.

Health: Claw's ready probe should not require a live vendor
completion (that makes deploys depend on OpenAI). Ready =
process + SDK + maybe gateway reachability.

## What the platform gave us

Version 1 was a model call. Version "done" is still a short
orchestrator. The difference is not more Python in Sarah's
repo. It is:

- **Gateway** — one door, identity, traces started.
- **Model Service** — vendors, stream, retry, route, cache,
  named prompts.
- **Session Service** — this talk, and memories across talks.
- **Data Service** — indexes, hybrid search, isolation.
- **Tools and guardrails** — actions with credentials and
  policy, not keys in prompts.
- **Observability** — waterfalls, scores, cost by workflow.
- **Experimentation** — datasets, offline, online, A/B.
- **Workflow Service** — the container, the route, the job
  if you add async research later.

A script can fake any one of these for a demo. It cannot
give the fifth team the same path without copying Redis
glue. That was chapter 1. Claw is the existence proof.

You can still buy a vendor suite that implements these
boxes. You cannot skip **naming** them. If you cannot point
to where sticky A/B assignment lives, you do not have
experiments. You have a flag.

Compare Agents ch. 11: those notes teach you how a support
or RAG *agent* should think (layers, field tips). These
notes teach you where that agent **runs** so the next
assistant is not a second snowflake.

### What this chapter is not

It is not a new service. If you invented a ninth platform
daemon to "run Claw," reread chapter 8.

It is not a full multi-agent org chart. Hub-and-spoke and
teams remain Agents. Claw may *call* other workflows.

It is not a promise that assembly order in the context
section is the only correct one. It is a promise that
**someone** must own the budget.

## Check yourself

1. Draw Claw v1 vs "complete" Claw. Which boxes are
   platform services, and which file is Sarah allowed to
   keep small?
2. Turn 1: "I'm Priya." Turn 2 (same session): "What's my
   name?" Turn 3 (new session tomorrow): "What seat do I
   like?" (she said window last month). Which service
   answers 2 vs 3, and what goes wrong if both facts live
   only in a vector index?
3. Always-on RAG vs agentic RAG: pick a Claw utterance that
   should *not* search, and one that should search twice.
   What loop cap do you set so the second kind cannot run
   all night?
4. A user says "invite everyone to a 'quick sync.'" At
   which inspection point do you stop this, and why is a
   system-prompt sentence insufficient?
5. List the six context sources. You have 8k tokens left
   after the system prompt. Who loses first — history,
   chunks, or tool schemas — for (a) a long chat with a
   short factual question, (b) a first-turn policy
   question with ten tools registered?
6. Streaming vs sync: where do output guardrails run in
   each, and what user-visible failure appears if you
   yield raw tokens then filter at the end?
7. An 8s Claw turn: waterfall shows Data 6.5s, Model 0.4s,
   tools 0. You have no custom spans. What do you *not*
   need to add to the workflow, and what Dataset case
   might you harvest?
8. You reorder the prompt to fix tool choice. Why is
   shipping that in the same deploy as a new model and a
   new `k` a mistake, and what assignment key makes the
   A/B not scramble mid-thread?
9. `genai-platform deploy`: name two things that must
   *not* be in Claw's image (a provider key, a second
   team's search implementation) and where each actually
   lives.
10. After this folder, what do you still take from
    [agents ch. 11](../../agents/11-field-tips/) — in one
    sentence — that the platform will not invent for you?

Continue to the [topic index](../../INDEX.md).
