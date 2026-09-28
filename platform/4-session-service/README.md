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

This folder stays on the **Session Service** — contract, storage
backends, model-managed memories as a *platform* store, and token-budget
strategies the org can share.

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
  system prompt | tools | RAG chunks | memories
  rolling summary | recent verbatim messages | reserved output
```

The one sentence to remember: **the transcript is an append-only log;
memory is a mutable map; the window is a budget.** If you only have a
log, long chats forget the first constraint. If you only have a map,
you lose the last three turns of nuance. If you ignore the budget,
the model never sees either.

## What a session contains

"Session" means a JWT, a shopping cart, a WebSocket, and a chat log
depending on who you asked. The service needs a precise object it can
store and return.

A session is **one conversation** between a user (or tenant principal)
and *this* application: identity, timestamps, metadata, and an ordered
list of **messages** in the same shape the Model Service already speaks.

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
tool_calls, tool results, assistant prose. Store all of them. If the
next generate() has a hole, the model invents a tool result.

```
  user: "start a return for order A14"
  assistant: tool_calls[check_order]
  tool: {status: delivered, opened: true}
  assistant: "Opened items have a restocking fee; continue?"
```

Four messages, one user-visible beat. `AddMessages` should take the
batch so a crash cannot persist the tool call without the tool row.

Use the OpenAI-like message object on purpose. Model Service adapters
already translate *out*. Session should keep that same canonical form
so you avoid a second translation layer and the bugs that come with it.

Metadata is how you debug "only iOS users lose history" without
parsing prose. Keep it small — channel, locale, product surface. If a
fact must survive truncation, it belongs in **memory** (later in this
chapter), as a keyed entry rather than a blob on the session row.

## The session service contract

Workflows need "get or start," "list this user's threads," "append a
turn," "read history," and "forget." Without a shared contract, five
slightly different REST apps appear.

Five RPCs, two groups:

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

**ListSessions** is how a UI shows "yesterday's return." It is not how
the model reads 400 threads. Return summaries (id, updated_at, title or
first user line), not full transcripts. **AddMessages** is batched: a
tool-using turn is N messages with one write. **GetMessages** is
where pagination *and* later strategies (truncate, hierarchical)
attach. `limit` / `before_position` are for UIs and for "give me
raw." `token_budget` + `strategy` are for model context. Keep those
paths separate — humans debugging a complaint want bytes, while model
context wants a budgeted view. **DeleteSession** is a product and
compliance verb (retention, "forget this chat"). Whether memories
cascade is an application policy you document; leave it explicit.

Authz: a user lists *their* sessions. A workflow identity still needs
authorization checks — guessing a UUID is not enough to call
`GetMessages`.

## Storage abstraction

The proto does not name Postgres. That is intentional.

### Choosing the right database

One team's audit log is another team's hot cache. Forcing everyone onto
the platform team's favorite DB either overpays (a full VM for a game
that can lose a room) or under-complies (Redis for a clinic).

Pick per **failure mode**, and name what you are accepting:

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
different apps. Register those backends with the Session Service so
the org still has one contract. A teaching platform picks Postgres and
documents the ABC so a later Redis implementation is a class, not a
fork of the servicer.

### The storage interface

gRPC handlers with SQL in them make swapping Redis a rewrite of the
service. Put persistence behind `SessionStorage`: the same five
operations, Python in / Python out. Postgres talks SQL. Redis talks
commands. Tests talk an in-memory fake. The servicer does not import
`psycopg2`.

```
  class SessionStorage(ABC):
      get_or_create_session(user_id, session_id=None) -> Session
      list_sessions(user_id) -> list[Session]
      add_messages(session_id, messages) -> None
      get_messages(session_id, ...) -> list[Message]
      delete_session(session_id) -> None
```

Keep it small. Analytics belongs on a different store or a replica.
If the ABC grows a `query_sql`, the abstraction has already leaked.

## Implementing the Session Service backend

Teaching implementation: Postgres. Patterns transfer: map methods to
storage verbs, transactions, connections.

### The database schema

Two tables. A single JSON column on the session row will hurt later.

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

Why separate rows instead of one JSONB array on `sessions`? Append
would rewrite a large row, pagination would become a JSON slice in the
app, and a single huge document locks the session row during a tool
turn. Rows scale; blobs surprise you at 5k messages.

Tool fields are JSON because the Model Service already treats them
as structured. Keep `tool_calls` out of free-text `content` so the next
turn can replay them faithfully.

### The storage implementation

`get_or_create` is easy to get racy: two first messages create two
sessions for one user who passed no id.

Use one transaction: if `session_id` given, SELECT and authorize; else
INSERT returning id. Unique constraints beat check-then-act.
`add_messages` inserts N rows in one transaction so you never persist
a tool call without its result (or you persist neither).

Connection story: pool, not a connect() per RPC. Map DB errors to
gRPC statuses (`NOT_FOUND`, `PERMISSION_DENIED`). The rest of the
platform should see those statuses, with database-specific exceptions
kept inside the storage layer.

`list_sessions` is `WHERE user_id = $1 ORDER BY updated_at DESC`
with a cap. `delete_session` is a transaction that removes messages
(or relies on CASCADE) and then the session row. If memories are
session-scoped and policy says cascade, that is the same
transaction; if not, leave them.

An in-memory implementation of the same ABC is how CI stays honest
without Docker-in-Docker. If tests only pass on Postgres, you do not
have an ABC; you have a leak.

### The gRPC service implementation

Thin. Bytes in, storage method, bytes out. Keep summarization policy
out of the first cut of the servicer — or, if
`GetMessages(strategy=...)` lives here, it **calls** helpers and
leaves SQL for summaries out of the protobuf layer.

```
  gateway  -->  SessionServiceServicer  -->  SessionStorage
                      (translate)                (persist)
```

The workflow speaks `platform.sessions`. It never speaks SQL.

## Integrating with the SDK

Sam will not write protobuf. If the SDK is awkward, he will `print`
history into the prompt again.

Ship `SessionClient(BaseClient)` with `service_name="sessions"`.
Methods: `get_or_create`, `list`, `add_messages`, `get_messages`,
`delete`. Python lists in, Python objects out. Same lazy channel as
`ModelClient`.

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
turn that contained them. This chapter's job is **a store with an
API** so every workflow does not invent `memories` JSON in session
metadata.

### Where memory lives

Tie facts only to a session and the discount dies when the chat ends.
Tie them only to a user and a shopping "navy case" haunts next season.
Tie them to nothing and you have a global brain that fails ACL.

Every entry has `user_id` and **optional** `session_id` (provenance,
and a filter when you *want* session scope).

```
  user 123
    memories (user-scoped):  discount_tier=enterprise
    session A memories:      shopping: wants navy case   (TTL?)
    session B memories:      (none extra)
```

Cascade-on-delete of a session is a **product** flag. Clinic: leave
user memories. Retail experiment: cascade session-scoped ones. GDPR
"forget me" is `clear_user_memory`, which clears the user map; deleting
the last session alone is a different verb with a different scope.

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
update in place — unlike messages, memory is mutable. "They are on the
enterprise plan now" overwrites. If you need history of changes, that
is audit, a separate table, with its own retention rules.

Keep keys small and columnar in spirit: `discount_tier`,
`preferred_channel`, `open_ticket_id`. A catch-all `notes` key turns
memory into a document store and loses queryability.

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
*once*. Write the uniqueness rule down so the first implementer does
not invent a private one.

### SDK integration

`platform.sessions.save_memory` / `get_memory`. JSON encode on the
wire (protobuf wants bytes or string); decode to dicts for Sam.
Handlers should decode JSON rather than pickle Python objects.

Load memories **before** generate; pass them as compact system lines
or as a tool the model can query. Saving **during** generate is a
tool call that Execute handles elsewhere; here you persist the resulting
entry.

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
never call `save_memory`, the fact lives only in the transcript and
dies under truncation. If you save every sentence, you have a second
transcript with worse queryability. Prefer keys the tool schema names
and values you would put in a column.

## Managing context windows

Faithful storage is not faithful prompting. Windows fill. System
prompts, tools, retrieved chunks, memories, and the reserved completion
all eat tokens before history gets a bite.

The platform should offer **strategies**. The workflow should choose.
A password-reset bot and a clinic bot should each name their strategy
and budget rather than share a hidden global `[-20:]`.

### The token budget problem

People say "the model has 128k." Then they add a 4k persona, 40 tool
schemas, eight retrieved chunks, a memory dump, and still expect 200
turns of chat.

Treat the window as a **pie**. History gets the remainder. Count with
the **same tokenizer family** the target model uses, or you will be
confidently wrong.

```
  | system | tools | data chunks | memories | HISTORY | output reserve |
  |  2k    |  3k   |     4k      |    1k    |   ???   |      2k        |
```

Session Service owns assembling HISTORY (and often memories). Other
services own their slices. Fighting over the pie without numbers is how
you blow the context and blame the vendor.

### Simple truncation

You need a default that is fast and has no extra model spend.

Walk **backwards** from the latest message, keep while tokens remain,
drop the rest. Good for independent FAQs. Painful when "I'm allergic"
lands in turn 1 and a prescription question lands in turn 40.

```
  kept:     [m31 ........ m50]     dropped: [m1 .. m30]
  m1 was "our enterprise discount" --> model no longer knows
```

Always keep a leading **system** message if it is part of the list;
dropping it is a different bug from dropping old user turns. Prefer
not to split a tool-call group (call without result). Token count
belongs in the service (or a shared tokenizer library), so twelve
workflows do not disagree about whether "return policy" is two tokens
or four.

### Compressing history with summarization

Truncation throws away the discount. You still cannot fit verbatim
turns 1–30.

LLM-compress the old slice into a short labeled block, keep recent
turns verbatim.

```
  [SUMMARY of m1-m30: enterprise discount; opened laptop; wants label]
  [m31 .... m50 verbatim]
```

Mark it as a summary so the model does not treat it as a quote.
Cache the summary keyed by the range of message ids; do not
re-summarize an idle thread every click. Use a **cheap** model for
this hop via cost-aware routing. Wrong summaries are false memories —
put safety-critical or commercial facts (`discount_tier`) in
**MemoryEntry** too, so they survive even when the paragraph softens
"enterprise" into "some discount." A summary that loses that word is a
billing incident wearing a helpful tone.

### Hierarchical memory

One strategy is a blunt instrument. Production wants layers.

Three tiers, different fidelity:

```
TIER 1  model-managed memories     structured, must not drop
  TIER 2  rolling summary            older narrative, lossy
  TIER 3  recent verbatim            last N turns, full nuance
```

Assemble in that order (plus retrieved document chunks from elsewhere).
If the pie is still too small, shrink tier 3 first, then re-summarize
tier 2, and keep tier 1 intact.

This is the default worth offering on `GetMessages(strategy=
"hierarchical", token_budget=...)`.

### Retrieval-augmented memory

"What did you recommend for headaches last time?" lives in **another**
session, months ago. Hierarchy inside *this* thread cannot see it.

Treat past sessions as a corpus. Search (usually via indexes of session
text, user-scoped) and insert hits labeled as *prior conversation*,
with a clear label so the model treats them as past context rather than
this turn's user voice.

```
  current session  ----hierarchical assemble---->
  past sessions    --search(user_id, source=sessions)-->
                         snippets into context
```

A retrieval miss looks like forgetting. Say so in the persona
("if no prior hit, ask"). Leave room for the model to admit the window
does not contain a life history.

This chapter's job is that the **index and the call** exist so
workflows can retrieve prior turns without each team embedding its own
search stack.

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
1 → include hierarchy or memory. Yes to 2 → memories. Yes to 3 → search
over past sessions.

```
  get_messages(..., strategy="truncate", limit=20)
  get_messages(..., strategy="hierarchical", token_budget=15000)
  search(query, filters={user_id, source: sessions})
```

The workflow still **composes**. The platform still **stops every
team from writing a worse `[-20:]`**. Trigger retrieval when the
utterance smells like "last time" if you want to save search spend;
measure misses.

## What this service is not

PDFs and published policies belong in the organizational knowledge
store. "Enterprise discount" said in chat is Session (memory or
transcript). Stuffing that sentence into a vector index of manuals is
how it rotates out of retrieval and becomes a hallucination.

Semantic / episodic / procedural memory as a research taxonomy is a
separate concern. You may *implement* those as keys or as extra stores
later. This chapter's contract is messages + entries + budget.

Summarization *calls* the Model Service. Session does not generate
user-facing replies.

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
10. Name one fact that belongs in a MemoryEntry key and one that
    belongs only in the transcript. What goes wrong if you swap them?
