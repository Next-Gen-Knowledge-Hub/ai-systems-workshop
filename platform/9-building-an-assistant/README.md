# 9. Building an AI assistant

Companion notes for **Chapter 9** of *Designing AI Systems* (Suhas Suresha and Dewang Sultania; Manning MEAP).

This chapter assembles the platform into one assistant, Claw. Sarah's code stays a short
orchestrator. The services do the product work: a named prompt, this conversation, memories, company
documents, tools under policy, a trace, an experiment, and a deploy. Afterward you can walk one
message through that loop and name which service owns each beat: model, session, data, tools,
guardrails, observability, and workflow.

## 1. The blueprint

Claw is a personal workplace assistant. People reach it from Slack or a web box. It remembers the
person, grounds answers in company documents, and takes actions — calendar, tickets, the web — under
policy. Traces show the work. Experiments decide what to keep. A workflow is how it ships. Write
that blueprint on one page before the first `chat()`: who the user is, which indexes, which tool
namespaces, which policies, and whether the product streams.

* **The function is the top**: Every box under the function is a platform service. The application
  is the orchestrator. If Sarah's file is 2,000 lines of HTTP and prompt glue, she reimplemented the
  platform. The provider call is about 2% of the system, and it sits at the leaf.
* **Channels share a door**: Slack and the web app POST the same `api_path`. The gateway
  authenticates, starts the trace, and routes. A channel that invents its own session ids splits the
  assistant on the first day you add a second client.
* **Name the services on the page**: Indexes, tool namespaces, policies, sync versus stream. A
  drawing that cannot name them is a demo, not an architecture.

```
  Slack / web / API
          |
          v
     API gateway   (auth, trace, route)
          |
          v
  @workflow Claw
    1 input guardrails
    2 session + long-term memories
    3 assemble context
    4 loop: model --> tool calls --> tools, under arg policy
    5 output guardrails
    6 store the exchange; stream or return
          |
          +-- Model, Session, Data, Tools, Guardrails
          +-- Observability and Experimentation record the rest
```

## 2. Version one is a model call

Every project starts here, and it should say so. Claw v1 is `@workflow`, a system prompt registered
with the Model Service, `platform.models.chat(...)`, and the text returned. No memory. No retrieval.
No tools.

* **Already on the platform**: Retries, routing, cache, and provider fallback apply on that one
  call. You did not write them. The honest diagram is a single box under the function.
* **The product is the small part**: A thin SDK is not what makes v1 a toy. The product cannot
  remember and cannot act. The user says "as I said," and Claw has nothing to refer to. A longer
  prompt does not create a conversation.
* **Register `claw-assistant`**: The prompt has a name in the Model Service registry. Later versions
  and A/B tests point at that name. A 40-line prompt that exists only inside the handler turns every
  experiment into a diff of Python.

## 3. This conversation

Load or create a session, pass the history into the model call, and append the exchange afterward.
The store outlives the pod, so a conversation survives a deploy and a hop to another replica.

* **What the handler adds**: A `session_id` comes in. `get_or_create` loads or opens the session.
  `history` goes into `chat`. `append` stores the user turn and the assistant turn. The client sends
  the id back on the next turn: a cookie, a Slack thread map, or a JSON field.
* **One session per conversation**: A key of only `user_id` stuffs every chat into one transcript
  and blows the token budget. A new session on every message throws the history away, and you are
  back at version 1.
* **Recent turns, on purpose**: Truncation, a summary, and hierarchical memory start to matter once
  the chat outgrows a demo. Version 2 may send recent turns. It must stop before the provider
  rejects the payload for size.

```
  v1:  message --> Model --> text
  v2:  session.get --> Model(history) --> session.append
```

* **Priya, same session**: Turn 1 is "my name is Priya." Turn 2 is "what's my name?" If turn 2
  fails, stop. Retrieval will not repair a conversation you never stored.

## 4. Facts that outlive the transcript

A sliding window is this sitting. The next session starts empty unless you keep facts that should
survive the transcript. Model-managed memories work like this: the model, or a dedicated pass,
proposes what to keep, and the Session Service stores it for that user. "Prefers window seats."
"On-call for payments this week."

* **Two bundles from Session**: Recent messages, and a compact memory list. Both enter the context.
  Each memory is small. Together they are why the assistant feels like it knows the person.
* **Memories stay out of the document index**: "Priya prefers window seats" is not an HR PDF. A
  preference dropped into Data comes back as a retrieval miss, or not at all. A PDF dropped into
  memory comes back as biography.
* **A write policy**: Preferences and project nicknames are fair game. Passwords are not. Health
  details your compliance rules forbid are not. A guardrail can block the bad write later. Session
  still should not keep whatever the model happened to whisper.
* **Priya, next session**: Close the session. Open a new one tomorrow. Ask "what seat do I like?"
  She said window seats last month. If the answer fails, you have a chatbot with a long window.

## 5. Grounding, always-on and agentic

Memories are about the user. Documents are about the organization. With no retrieval, Claw answers
an HR question from pretraining, fluently. That is the dangerous kind of confidence.

1. **Always-on retrieval**: Call `data.search` or `hybrid_search` on every turn, place the top
   chunks in context, then generate. Use it when the turn is "what is the parental leave policy." It
   is short to write. It still searches on "hello," and on "schedule a meeting with Sam." It also
   sticks with a bad first query string, because the loop never gets a second search.
2. **Agentic retrieval**: Retrieval is a tool, or a step the model may request. The model can
   reformulate, search again, or skip. You pay latency and you take on loop risk. Messy questions
   get a better chance of a relevant chunk. The call is still Data, once or several times, from the
   tool loop.

```
  always-on                         agentic
  every turn: search                model may search(query')
  k chunks always in context        0..n searches
  one hop                           extra hops, extra traces
```

* **Indexes stay separate**: `hr.policies` is not the same index as `eng.runbooks`. Isolation is how
  Claw avoids citing a sealed memo. A filter such as `audience=employee` belongs on the search call.
  A sentence in the prompt that says "only use public docs" does not enforce it.
* **Support the claim**: A citation the chunks do not support still needs a score, and sometimes a
  critic. This chapter puts the chunks in the window and lets the loop retrieve again. The score
  lands with the trace.

## 6. Tools and the loop that uses them

A wrong sentence is embarrassing. A calendar invite to 500 people is an incident. Tools are how Claw
changes the world without holding a vendor key in the workflow.

* **A kit**: Register `claw.calendar.list`, `claw.tickets.create`, `claw.web.search`, and
  `claw.knowledge.search` when retrieval itself is a tool. Discover that kit, not every tool in the
  company. Pass the schemas to the model. On tool calls, `tools.execute`, append the observations,
  and call the model again.
* **The workflow owns the loop**: The shape is ReAct. The function decides to continue or stop. The
  Tool Service is the socket that validates, authorizes, and runs the call. OAuth stays out of the
  handler. So does `requests.post` to Jira with a token from the environment.
* **Caps inside the timeout**: Set a max iteration count, a max number of tools per turn, and a
  timeout. The workflow timeout is the outer envelope. The loop still needs its own counter. An
  uncapped loop is how a weekend of research becomes Monday's invoice.
* **Reads, then writes**: Expose read tools before write tools on a general assistant. Anything that
  emails, spends, or invites waits on a confirmation policy.
* **One agent is a valid Claw**: Specialists, a hub, or a team are a composition of workflows, or a
  graph inside this process. They are not an extra platform service.

```
  model --> text --> done
        --> tool_calls --> validate args --> execute --> observe --> model
                          (stop on text or on the iteration cap)
```

## 7. Four places policy can stop the turn

Claw without policy is a friendly name on a liability. "Invite the whole company" as a joke, medical
advice, a performance complaint pasted into a public ticket: those show up. Plan the inspection
points before they do.

1. **Input**: Jailbreaks, personal data, and disallowed topics, before you spend a retrieval or a
   model call. A block here skips search and skips generation.
2. **Tool arguments**: After the model proposes a call, before Execute. A schema check is the start.
   "This calendar is the authenticated user's." "The invite list is at most N." "The ticket project
   is on the allow list."
3. **Output**: Leakage, toxicity, and a required disclaimer, before Slack or the web client shows
   the text.
4. **Behavioral**: Which tools this application is allowed to discover, and which irreversible
   writes require confirmation.

* **The prompt instructs, the service enforces**: "You must not…" belongs in the system prompt as
  guidance. The Guardrails Service and the tool policy are what actually stop the call.
* **A refusal the user can read**: False positives block real work, so measure them. A silent drop
  looks like a hang. Return the refusal the policy produced. A model that improvises an apology will
  apologize for the wrong thing.
* **Internal users still probe**: A healthcare-adjacent Claw treats input policy as required. An
  internal-only Claw still needs behavioral rules. People on the same SSO jailbreak for fun.

## 8. The context window is a budget

Pieces have been landing in the window all along. At some length the window is a budget with owners.
Session would keep every turn. Data would keep every chunk. Tools would keep every schema. The
prompt registry would keep a novella. Context engineering is the workflow deciding what fits.

```
  context window
    1 system prompt     Model registry, versioned
    2 long-term memory  Session
    3 retrieved docs    Data
    4 tool schemas      the kit from discovery
    5 recent turns      Session, truncated or summarized
    6 current message   the user
```

* **How a bad mix fails**: Too little history and the assistant loses the thread. Too many documents
  and it invents links between unrelated PDFs. Too many tool schemas and the tokens go to JSON the
  model will never call. Irrelevant memories feel creepy, or they personalize the wrong fact.
* **A teaching assembly order**: The system prompt first, stable and versioned, including when to
  use a tool and when to search. Memories next, few of them, under a hard cap of N facts. The
  current user message has to fit; when it does not, reject it or chunk the user's own text. Do not
  clip a legal question and hope. Then the schemas for this kit. Then retrieved chunks under a token
  cap, with citations. History receives whatever budget is left.
* **The prompt only claims wired powers**: The system prompt grows when a capability appears.
  Version 1 does not mention tools you have not connected. "You can refund," with no refund tool, is
  how a refund gets hallucinated.
* **The split is on the trace**: You should be able to read how many tokens were prompt, tools,
  documents, and history. That split is what you tune. An assembly order other than the one above
  can be right. What cannot be optional is that some part of the workflow owns the cut.

## 9. The loop, including a stream

One flow. The boxes are platform calls. The branches are what make Claw an agent rather than a
straight pipeline. Keep it in one workflow function until a stage needs a different scale or a GPU.
Heavy research can be a child workflow later. It does not have to be the first design.

```
  message
     |
     v
  input guardrails --block--> refusal
     |
     v
  session + memories --> assemble the six sources
     |
     v
  model <--------------------------------------+
     |                                         |
     +-- text --> output guardrails            |
     |              |                          |
     |              v                          |
     |         append session --> reply        |
     |                                         |
     +-- tool calls --> argument policy --block+
                         |
                         v
                       execute --> observe ----+
                       (iteration cap ends the loop)
```

* **Stop before spend**: An input block does not search and does not generate.
* **The tool branch**: Validate each proposed call, execute, append the observations as messages,
  and call the model again. At the iteration cap, return a controlled failure — "I could not finish
  this" — rather than the last half-written tool thought.
* **Two stores, two paths**: Tool observations go into the session when the next user turn needs
  them. They are history. Memories are written only through the memory path. A transcript you hope
  will still be there next month is not a memory.

Streaming is the same loop with a different edge. Set `response_mode="stream"` and call
`chat_stream`. A single JSON body at the end is easier to compose and slower to feel.

* **Tokens versus tool calls**: On each iteration, content tokens yield to the client immediately.
  Tool-call deltas stay in a buffer. The user should not see broken JSON arguments. Text runs
  through output policy and finishes the turn. Tool calls execute, then the loop continues.
* **`filter_output` versus `filter_output_stream`**: Sync Claw can filter the full string. Stream
  Claw cannot hold the entire answer without abandoning the reason you streamed.
  `filter_output_stream` yields by sentence, or by another span long enough for the policy,
  buffering only that span. Some leaks are invisible until the sentence ends. A few hundred
  milliseconds of buffer is the compromise. Raw tokens must not pass a policy that exists to stop
  them.
* **A sentinel, then close**: A mid-stream error sends a sentinel and closes. Half an answer
  followed by silence is a worse product than a stop the user can see. Filtering only after every
  token has already been yielded shows the sentence the policy was there to catch. Holding the
  entire stream and filtering at the end has rebuilt synchronous mode.

## 10. Traces, then one experiment

In production Sarah's questions are operational. Why did this reply take 8 seconds when most take 3?
What did this user cost today? Which searches returned junk? Is quality moving, or did people stop
filing bugs?

* **Spans come with the calls**: She does not wrap each `platform.*` call by hand. Model records
  tokens, cost, time to first token, and the model id. Data records the query and the hit counts.
  Tools record duration and status. Guardrails record the policy and the action. The waterfall is
  the architecture diagram with times filled in.
* **Read the slow box**: A 5 second Data span with 0 hits is a grounding failure. It is not a slow
  model. A cheap model with a bad helpfulness score is a routing problem. A tool span that fails and
  then succeeds on retry is an adapter flake. An 8 second turn that shows Data at 6.5s, Model at
  0.4s, and no tool span does not need new workflow spans. It needs a look at that search, and the
  trace is a candidate for a dataset.
* **Glue and access**: A homemade rerank or a rules engine needs a custom span, or the waterfall
  shows a gap. Generations contain the user's text. The list of people who can open a trace is part
  of shipping Claw.

Observability answers what happened. An experiment answers whether a change helped. Claw calls
`web.search` when the internal index was the right tool. The hypothesis: tool guidance sits too late
in the prompt. The treatment: a new prompt version that puts tools first. Taste is not a ship
decision.

* **Control and treatment**: Control is the current prompt name and version. Treatment is
  `claw-assistant-v5-tools-first`. Assignment is a sticky hash on `user_id`, so one person does not
  switch variants mid-thread. Metrics are tool-selection accuracy, helpfulness, cost, and latency.
  Stop when you have enough traffic, or when a guard fires — a quality collapse aborts the run.
* **One knob in the deploy**: Changing the prompt, the model, and `k` together leaves nothing you
  can promote on purpose. Online scoring should already be sampling Claw. The experiment slices
  those scores by variant. Run the offline dataset first when a bad answer would hurt someone. Use
  the A/B when the open risk is that real people do not phrase things like the file.
* **Move the pointer**: Point the workflow's `system_prompt_name`, or the target pointer, at the
  winner. Deprecate the loser. A registry of undeprecated prompts becomes a museum of guesses.

## 11. Deploy the workflow you meant

The decorator on Claw is the deploy contract. Name it. Set `api_path` to `/claw/chat`. Set the mode
to `stream` if that is the product. Set replicas, CPU, and memory. Set a timeout that covers a few
tool hops and refuses an unbounded research loop.

* **`genai-platform deploy claw.py`**: Build the image, register, deploy, wait until ready, register
  the gateway route. Slack and the web client call that path. They do not open a shell on a pod.
* **Keys stay in the services that use them**: Tool credentials are references on the tool, not
  environment variables baked into Claw's image. Provider keys live in the Model Service.
  `OPENAI_API_KEY` inside the Dockerfile means the workflow is calling a vendor on its own. Another
  team's search code stays in that team's workflow. Claw calls it. Copying it into this image copies
  their dependencies and misses their next fix.
* **Code ships as an image, prompts ship as a target**: Roll back by selecting the previous workflow
  version. Hand-editing the live prompt has no version to return to. When inference reads the prompt
  name on each request, a prompt-only change is a registry update and does not need a new image.
  Code and dependencies need a rebuild. Shipping both in one move publishes yesterday's code beside
  tomorrow's prompt, and the trace will not tell you which one you are blaming.
* **Readiness is the process**: The ready probe does not call the vendor for a completion. A probe
  that does will fail the deploy whenever the provider blips. Ready means the process is up, the SDK
  is constructed, and, if you choose, the gateway is reachable.

## 12. What the short function is standing on

Version 1 was a model call. The assistant you would actually leave running is still a short
orchestrator. The new mass is not more Python in Sarah's repository.

* **Gateway**: One door. Identity. The trace starts here.
* **Model Service**: Vendors, streaming, retry, routing, cache, named prompts.
* **Session Service**: This conversation, and memories across conversations.
* **Data Service**: Indexes, hybrid search, isolation between indexes.
* **Tools and guardrails**: Actions that carry credentials and policy. The keys are not in the
  prompt.
* **Observability**: The waterfall, the scores, the cost of this workflow.
* **Experimentation**: Datasets, offline runs, online scores, A/B tests.
* **Workflow Service**: The container, the route, and a job row if a later research path is async.

* **One path, shared**: A script can imitate any single box for a demo. The fifth team cannot reuse
  that script without copying the glue. Claw is the proof the path exists once. You can buy a suite
  that implements the same boxes. You still have to point at where sticky assignment lives. A
  boolean flag is not an experiment.
* **No extra runtime**: A new platform process whose job is "run Claw" repeats the Workflow Service.
  A hub or a team is a later design: Claw calls other workflows. That call is optional, and it does
  not add a service.
* **Someone owns the budget**: The six-part order is a teaching order. Another order can fit another
  product. An unowned window fills with whichever source is largest.

## Check yourself

1. Draw Claw v1 next to Claw with the full loop. Which boxes are platform services, and which file
   is Sarah's to keep short?
2. Turn 1, same session: "I'm Priya." Turn 2: "What's my name?" Turn 3, a new session tomorrow:
   "What seat do I like?" She said window seats last month. Which store answers turn 2, which
   answers turn 3, and what fails if both facts live only in a document index?
3. Name a Claw utterance that should not search, and one that should search twice. What cap stops
   the second kind from running all night?
4. The user says "invite everyone to a quick sync." At which inspection point do you stop it, and
   why does a sentence in the system prompt fail to stop it?
5. List the six context sources. After the system prompt, 8,000 tokens remain. For a long chat with
   a short factual question, and for a first-turn policy question with ten tools registered, which
   of history, chunks, and tool schemas gives up space first?
6. Where does output policy run for a sync reply, and where does it run for a stream? What does the
   user see if you yield raw tokens and filter only at the end?
7. An 8 second turn shows Data at 6.5s, Model at 0.4s, and no tool span. You added no custom spans.
   What do you not need to add to the workflow, and what about that trace makes it a dataset case?
8. You reorder the prompt so Claw picks the internal index. Why is shipping that reorder in the same
   deploy as a new model and a new `k` a mistake, and which assignment key keeps one user on one
   variant for the whole thread?
9. Name two things that must stay out of Claw's image — a provider key, and another team's search
   implementation — and where each one actually lives.
10. What does a running Claw demonstrate that a diagram of the same boxes does not?
