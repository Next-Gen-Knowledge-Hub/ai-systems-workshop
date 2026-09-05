# 4. The Session Service

Companion notes for **Chapter 4** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

Chapter 1 split **context-aware intelligence** in two. This chapter is
the conversational half: who is talking, what was said, what the model
chose to remember, and what still fits in the next call. Skip it and
every team owns a Redis of chat logs (or a JSON file), truncation is a
`messages[-20:]`, and Maria's enterprise discount dies at message 21.

The Agents track is a different book. Memory *forms*, MCP memory,
agent-shaped RAG: [agents ch. 6](../../agents/6-memory-and-rag/).
Mention that folder. Do not rewrite it. This folder stays on the
**Session Service** — contract, storage backends, model-managed
memories as a *platform* store, token-budget strategies the org can
share.

## The mental model

```
  workflow
     |
     +-- get_or_create(user, session?)
     +-- add_messages / get_messages(strategy, budget)
     +-- save_memory / get_memory
     v
  SessionClient  -->  gateway  -->  Session Service
                                         |
                                         v
                              SessionStorage (ABC)
                                         |
                    +--------------------+--------------------+
                    v                    v                    v
               Postgres              Redis               Dynamo / ...
               (audit)               (speed)             (serverless)

  ONE TURN'S CONTEXT WINDOW (Model Service still generates)
  ---------------------------------------------------------
  system prompt | tools | RAG chunks (Data) | memories
  rolling summary | recent verbatim messages | reserved output
```

The one sentence to remember: **the transcript is an append-only log;
memory is a mutable map; the window is a budget.** If you only have a
log, long chats forget the first constraint. If you only have a map,
you lose the last three turns of nuance. If you ignore the budget,
the model never sees either.

Truncation vs summary vs hierarchy vs retrieval:
[`TRADEOFFS.md`](../../TRADEOFFS.md) (memory table). Use this folder
for the *service* that implements those rows. Documents that are not
chat belong in [chapter 5](../5-data-service/).

## What a session contains

**Problem** — "Session" means a JWT, a shopping cart, a WebSocket, and
a chat log depending on who you asked. The service cannot store a fog.

**Solution** — A session is **one conversation** between a user (or
tenant principal) and *this* application: identity, timestamps,
metadata, and an ordered list of **messages** in the same shape the
Model Service already speaks.

```
  session
    id, user_id
    created_at, updated_at
    metadata          {channel: "ios", locale: "en", ...}
    messages[]
      role            system | user | assistant | tool
      content
      tool_calls[]    (assistant asked to act)
      tool_call_id    (tool result links back)
      created_at
```

A "simple" user turn is often **several** rows: user text, assistant
tool_calls, tool results, assistant prose. Store all of them or the
next generate() has a hole and the model invents a tool result.

```
  user: "start a return for order A14"
  assistant: tool_calls[check_order]
  tool: {status: delivered, opened: true}
  assistant: "Opened items have a restocking fee; continue?"
```

Four messages, one user-visible beat. `AddMessages` should take the
batch so a crash cannot persist the tool call without the tool row.

Use the OpenAI-like message object on purpose. [Chapter 3](../3-model-service/)
adapters already translate *out*. Session should not invent a second
canonical form.

Metadata is how you debug "only iOS users lose history" without
parsing prose. It is not a junk drawer for the entire CRM — if a fact
must survive truncation, it wants **memory** (later), not a blob on
the session row.

## The session service contract

**Problem** — Workflows need "get or start," "list this user's
threads," "append a turn," "read history," "forget." Five slightly
different REST apps appear.

**Solution** — Five RPCs, two groups:

```
  session lifecycle          messages
  ------------------         --------
  GetOrCreateSession         AddMessages
  ListSessions               GetMessages
  DeleteSession
```

**GetOrCreate** is one call so Sam does not race "get then insert"
on the first chat. Pass `session_id` to resume; omit it (or pass
null) to start. Return the id every time — clients should not mint
ids the service never saw unless you explicitly allow client ids
(then they must be unique and authz-checked).

**ListSessions** is how a UI shows "yesterday's return," not how the
model reads 400 threads. Return summaries (id, updated_at, title or
first user line), not full transcripts. **AddMessages** is batched: a
tool-using turn is N messages with one write. **GetMessages** is
where pagination *and* later strategies (truncate, hierarchical)
attach. `limit` / `before_position` are for UIs and for "give me
raw." `token_budget` + `strategy` are for model context. Do not make
the UI pager silently summarize — humans debugging a complaint want
bytes. **DeleteSession** is a product and compliance verb (retention,
"forget this chat"). Whether memories cascade is an application
policy, not a silent default.

Authz: a user lists *their* sessions. A workflow identity is not a
license to `GetMessages` on any UUID it guessed.

## Storage abstraction

The proto does not name Postgres. That is intentional.

### Choosing the right database

**Problem** — One team's audit log is another team's hot cache.
Forcing everyone onto the platform team's favorite DB either
overpays (VM for a game that can lose a room) or under-complies
(Redis for a clinic).

**Solution** — Pick per **failure mode**, not per fashion.

```
  class          examples        you gain              you pay
  -----          --------        --------              -------
  relational     Postgres,       crash-safe, SQL       schema + ops,
                 MySQL           audit, joins          migrations
  in-memory      Redis           sub-ms reads          durability,
                                                       eviction surprises
  document       Mongo, etc.     floppy documents      weaker invariants
  serverless     Dynamo, ...     scale-to-zero         query shapes,
                                                       vendor lock
```

Regulated chat (health, money): relational, PITR, who-touched-what.
Ephemeral game master: Redis, short TTL. Burst-and-idle: a managed
store that sleeps. TTL on shopping chats is a **feature**; TTL on
"enterprise discount" as a remembered fact is a bug — that fact
wanted user-scoped memory, not a 24h session row.

You may run **more than one** backend behind the same ABC for
different apps. You should not run four *unregistered* backends that
the Session Service does not know (chapter 1 sprawl). A teaching
platform picks Postgres and documents the ABC so a later Redis
implementation is a class, not a fork of the servicer.

### The storage interface

**Problem** — gRPC handlers with SQL in them. Swapping Redis means
rewriting the service.

**Solution** — `SessionStorage` ABC: the same five operations, Python
in / Python out. Postgres talks SQL. Redis talks commands. Tests talk
an in-memory fake. The servicer does not import `psycopg2`.

```
  class SessionStorage(ABC):
      get_or_create_session(user_id, session_id=None) -> Session
      list_sessions(user_id) -> list[Session]
      add_messages(session_id, messages) -> None
      get_messages(session_id, ...) -> list[Message]
      delete_session(session_id) -> None
```

Keep it small. "Also run analytics" is a different store or a
replica. If the ABC grows a `query_sql`, you have failed the
abstraction.

## Implementing the Session Service backend

Teaching implementation: Postgres. Patterns transfer: map methods to
storage verbs, transactions, connections.

### The database schema

Two tables, not one JSON column you will regret.

```
  sessions                         messages
  --------                         --------
  id PK                            id PK
  user_id                          session_id FK  ON DELETE CASCADE
  created_at                       role
  updated_at                       content
  metadata JSONB                   tool_calls JSONB
                                   tool_call_id
                                   created_at
                                   position      (order in the log)
```

**Sessions** are the handle: resume, list, ACL, last activity.
**Messages** grow. Index `(session_id, position)` or `(session_id,
created_at)`. Do not select `*` of a 10k-turn thread on every hop
without a limit — `GetMessages` is where the budget starts even
before summarization.

Why not one JSONB array on `sessions`? Append is rewrite of a large
row, pagination is a JSON slice in the app, and a single huge
document is how you lock the session row during a tool turn. Rows
scale; blobs surprise you at 5k messages.

Tool fields are JSON because the Model Service already treats them
as structured. Do not stringify tool_calls into `content` and hope.

### The storage implementation

**Problem** — `get_or_create` is easy to get racy: two first messages
create two sessions for one user who passed no id.

**Solution** — One transaction: if `session_id` given, SELECT and
authorize; else INSERT returning id. Unique constraints beat
check-then-act. `add_messages` inserts N rows in one transaction so
you never persist a tool call without its result (or you persist
neither).

Connection story: pool, not a connect() per RPC. Map DB errors to
gRPC statuses (`NOT_FOUND`, `PERMISSION_DENIED`). The rest of the
platform should not see `UniqueViolation`.

`list_sessions` is `WHERE user_id = $1 ORDER BY updated_at DESC`
with a cap. `delete_session` is a transaction that removes messages
(or relies on CASCADE) and then the session row. If memories are
session-scoped and policy says cascade, that is the same
transaction; if not, leave them.

In-memory implementation of the same ABC is how CI stays honest
without Docker-in-Docker. If tests only pass on Postgres, you do not
have an ABC; you have a leak.

### The gRPC service implementation

Thin. Bytes in, storage method, bytes out. No "business logic" about
when to summarize in the first cut of the servicer — or, if
`GetMessages(strategy=...)` lives here, it **calls** helpers, it does
not embed SQL for summaries in the protobuf layer.

```
  gateway  -->  SessionServiceServicer  -->  SessionStorage
                      (translate)                (persist)
```

This matches [chapter 2](../2-sdk-and-api/): the workflow never
speaks SQL. It speaks `platform.sessions`.

## Integrating with the SDK

**Problem** — Sam will not write protobuf. If the SDK is awkward, he
will `print` history into the prompt again.

**Solution** — `SessionClient(BaseClient)` with
`service_name="sessions"`. Methods: `get_or_create`, `list`,
`add_messages`, `get_messages`, `delete`. Python lists in, Python
objects out. Same lazy channel as `ModelClient`.

```
  session = platform.sessions.get_or_create(user_id, session_id)
  history, _ = platform.sessions.get_messages(session.id, limit=20)
  platform.sessions.add_messages(session.id, [user_msg, assistant_msg])
```

Return the session id on every path so the frontend can send it back.
Losing the id is how you fork a conversation every click.

Deadlines here are storage, not generation. A 30s model timeout is
the wrong default for `get_or_create`.

## Model-managed memory

The log is faithful and dumb. The model can be a better librarian.

The research picture (OS metaphor): context window as RAM, a store as
disk. The model **writes** facts it will need after RAM evicts the
turn that contained them. This chapter's job is not the paper. It is
**a store with an API** so every workflow does not invent `memories`
JSON in session metadata.

### Where memory lives

**Problem** — Tie facts only to a session and the discount dies when
the chat ends. Tie them only to a user and a shopping "navy case"
haunts next season. Tie them to nothing and you have a global brain
that fails ACL.

**Solution** — Every entry has `user_id` and **optional**
`session_id` (provenance, and a filter when you *want* session
scope).

```
  user 123
    memories (user-scoped):  discount_tier=enterprise
    session A memories:      shopping: wants navy case   (TTL?)
    session B memories:      (none extra)
```

Cascade-on-delete of a session is a **product** flag. Clinic: no.
Retail experiment: yes. GDPR "forget me" is `clear_user_memory`, not
"delete last session."

### The memory model

```
  MemoryEntry
    key          "discount_tier" | "preferred_channel" | ...
    value        JSON-able (string, list, object)
    user_id
    session_id?  provenance or scope
    created_at, updated_at
```

Keys are names the **tool schema** advertises (`save_memory`). Values
update in place — unlike messages, memory is not append-only. "They
are on the enterprise plan now" overwrites. If you need history of
changes, that is audit, a separate table, not a second log of chat.

Do not overload keys (`notes` as a novel). Then you have a document
and you wanted Data or a summary. A good key is something you would
put in a column: `discount_tier`, `preferred_channel`,
`open_ticket_id`.

### Extending the storage interface for memories

Same ABC, four more verbs:

```
  save_memory(user_id, key, value, session_id=None)
  get_memory(user_id, key=None, session_id=None)  # one or many
  delete_memory(user_id, key)
  clear_user_memory(user_id)                      # right to erasure
```

Postgres: a `memories` table unique on `(user_id, key)` or
`(user_id, session_id, key)` depending on scope rules you document
*once*. Do not make uniqueness "whatever the first implementer
thought."

### SDK integration

`platform.sessions.save_memory` / `get_memory`. JSON encode on the
wire (protobuf wants bytes or string); decode to dicts for Sam.
Handlers should not pickle Python objects.

Load memories **before** generate; pass them as compact system lines
or as a tool the model can query. Saving **during** generate is a
tool call (chapter 6 executes; here you persist).

### A complete workflow example

Three phases. The model decides what is worth a key; the platform
stores.

```
  memories = platform.sessions.get_memory(user_id)
  recent, _ = platform.sessions.get_messages(session_id, limit=20)
  messages = assemble(system + memories, recent, user_text)

  reply = platform.models.chat(messages, tools=["save_memory", ...])

  for call in reply.tool_calls:
      if call.name == "save_memory":
          platform.sessions.save_memory(user_id, **parse(call.args),
                                        session_id=session_id)
  platform.sessions.add_messages(session_id, turn_including_tools)
```

If you save only in prose ("I'll remember you're on enterprise") and
never call `save_memory`, you have theatre. If you save every
sentence, you have a second transcript with worse queryability.

Agents ch. 6 will talk semantic/episodic/procedural *forms*. This
example is the **platform write path**. Do not paste that chapter
here.

## Managing context windows

Faithful storage is not faithful prompting. Windows fill. System
prompts, tools, Data chunks, memories, and the reserved completion
all eat tokens before history gets a bite.

The platform should offer **strategies**. The workflow should choose.
A password-reset bot and a clinic bot should not share a hidden
global `[-20:]`.

### The token budget problem

**Problem** — People say "the model has 128k." Then they add a 4k
persona, 40 tool schemas, eight retrieved chunks, a memory dump, and
still expect 200 turns of chat.

**Solution** — Treat the window as a **pie**. History gets the
remainder. Count with the **same tokenizer family** the target model
uses, or you will be confidently wrong.

```
  | system | tools | data chunks | memories | HISTORY | output reserve |
  |  2k    |  3k   |     4k      |    1k    |   ???   |      2k        |
```

Session Service owns assembling HISTORY (and often memories). Data
Service owns chunks. Fighting over the pie without numbers is how
you blow the context and blame the vendor.

### Simple truncation

**Problem** — Need a default that is fast and has no extra model
spend.

**Solution** — Walk **backwards** from the latest message, keep while
tokens remain, drop the rest. Good for independent FAQs. Bad for
"I'm allergic" in turn 1 and a prescription question in turn 40.

```
  kept:     [m31 ........ m50]     dropped: [m1 .. m30]
  m1 was "our enterprise discount" --> model no longer knows
```

Always keep a leading **system** message if it is part of the list;
truncating it is a different bug. Prefer not to split a tool-call
group (call without result). Token count belongs in the service (or
a shared tokenizer library), not in twelve workflows that disagree
about whether "return policy" is two tokens or four.

### Compressing history with summarization

**Problem** — Truncation throws away the discount. You still cannot
fit verbatim turns 1–30.

**Solution** — LLM-compress the old slice into a short labeled
block, keep recent turns verbatim.

```
  [SUMMARY of m1-m30: enterprise discount; opened laptop; wants label]
  [m31 .... m50 verbatim]
```

Mark it as a summary so the model does not treat it as a quote.
Cache the summary keyed by the range of message ids; do not
re-summarize an idle thread every click. Use a **cheap** model
([chapter 3](../3-model-service/) routing) for this hop. Wrong
summaries are false memories — put safety-critical or commercial
facts (`discount_tier`) in **MemoryEntry** too, not only in the
paragraph. A summary that says "some discount" when the user said
"enterprise" is a billing incident wearing a helpful tone.

### Hierarchical memory

**Problem** — One strategy is a blunt instrument. Production wants
layers.

**Solution** — Three tiers, different fidelity:

```
TIER 1  model-managed memories     structured, must not drop
  TIER 2  rolling summary            older narrative, lossy
  TIER 3  recent verbatim            last N turns, full nuance
```

Assemble in that order (plus Data chunks from chapter 5). If the pie
is still too small, shrink tier 3 first, then re-summarize tier 2,
never silently drop tier 1.

This is the default worth offering on `GetMessages(strategy=
"hierarchical", token_budget=...)`.

### Retrieval-augmented memory

**Problem** — "What did you recommend for headaches last time?"
lives in **another** session, months ago. Hierarchy inside *this*
thread cannot see it.

**Solution** — Treat past sessions as a corpus. Search (usually via
**Data Service** indexes of session text, user-scoped) and insert
hits labeled as *prior conversation*, not as this turn's user voice.

```
  current session  ----hierarchical assemble---->
  past sessions    --Data.search(user_id, source=sessions)-->
                         snippets into context
```

Retrieval miss looks like forgetting. Say so in the persona
("if no prior hit, ask"). Do not pretend the window contains a life
history.

Agents ch. 6 is how an *agent* decides to retrieve again. Here: the
**index and the call** exist so workflows can do it without each
team embedding Redis search.

### Putting it together

No universal winner. Match the conversation:

```
  independent FAQs          truncate
  one-session shopping      hierarchical, no cross-session
  clinic / long support     memories + hierarchical + retrieve
  coding thread             retrieve past solutions; weight recent
```

How to choose: do users cite earlier turns? Is there a fact that
must never fall out (safety)? Do they cite **other days**? Yes to
1 → don't only truncate. Yes to 2 → memories. Yes to 3 → Data on
sessions.

```
  get_messages(..., strategy="truncate", limit=20)
  get_messages(..., strategy="hierarchical", token_budget=15000)
  platform.data.search(query, filters={user_id, source: sessions})
```

The workflow still **composes**. The platform still **stops every
team from writing a worse `[-20:]`**. Trigger retrieval when the
utterance smells like "last time" if you want to save search spend;
measure misses.

## What this service is not

Not the Data Service. PDFs and policies are [chapter 5](../5-data-service/).
"Enterprise discount" said in chat is Session (memory or transcript).
Stuffing that sentence into a vector index of manuals is how it
rotates out of retrieval and becomes a hallucination.

Not the agent memory taxonomy (semantic / episodic / procedural) as
a research program — Agents ch. 6. You may *implement* those as
keys or as extra stores later. This chapter's contract is messages +
entries + budget.

Not the Model Service. Summarization *calls* it. Session does not
generate user-facing replies.

## See also

Same words, different job — do not merge.

- **[agents ch. 6 — Memory and RAG](../../agents/6-memory-and-rag/)**
  How one agent retrieves and remembers. This folder is how the org
  stores turns and budgets tokens.
- [`TRADEOFFS.md`](../../TRADEOFFS.md) — truncation / summary /
  hierarchy / retrieval.
- Knowledge indexes: [chapter 5](../5-data-service/).

## Check yourself

1. A user turn with one tool call becomes how many messages? What
   breaks next turn if you persist only the final assistant prose?
2. Why do Session messages use the same role/content shape as the
   Model Service? Name a conversion bug you avoid.
3. GetOrCreate vs get-then-insert: what race do you lose without a
   transaction or equivalent constraint?
4. Clinic vs game vs idle-then-spike shop: pick a storage class for
   each and the failure you are explicitly accepting.
5. Should deleting a session delete memories? Give one product that
   says yes and one that says no. Which RPC still must exist for
   "forget this user"?
6. Draw the context pie for a 32k model with a fat tool list. Who
   owns each slice? What happens if history is "whatever is left"
   with no reserve for the completion?
7. Truncation dropped "enterprise discount" in turn 1. Which two
   mechanisms in this chapter would have saved it, and how do they
   fail differently (summary error vs retrieval miss vs never saved)?
8. Hierarchical vs retrieval-augmented memory: which one helps
   "as I said twenty turns ago in *this* chat," and which helps
   "last spring's visit"?
9. Why might summarization run through the Model Service with
   cost-aware routing instead of the same frontier model as chat?
10. This workshop keeps Agents and Platform separate. After this
    folder, what do you still open
    [agents ch. 6](../../agents/6-memory-and-rag/) for — and what
    would be a mistake to copy into Session Service?

Continue to [The Data Service](../5-data-service/).
