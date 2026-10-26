# 4. The Session Service

Companion notes for **Chapter 4** of *Designing AI Systems* (Suhas Suresha and Dewang
Sultania; Manning MEAP).

This chapter establishes the Session Service. The transcript is an append-only log, memory
is a mutable map, and the context window is a budget. Afterward you can name what one
conversation stores, the calls a workflow uses, which storage failure you accept, and
whether Maria's enterprise discount survives by a memory key, a summary, or a search over
past chats. Without the service, every team owns a private chat log, truncation is
`messages[-20:]`, and
the discount dies at message 21.

## 1. What a session contains

A session is one conversation between a user (or tenant principal) and this application. The
word also means a JWT, a cart, or a WebSocket. The service stores one object.

```
  session
    id, user_id
    created_at, updated_at
    metadata        {channel: "ios", locale: "en"}
    messages[]
      role          system | user | assistant | tool
      content
      tool_calls[]  (assistant asked to act)
      tool_call_id  (tool result links back)
      created_at
```

* **Canonical messages**: `role`, `content`, and the tool fields, in the same shape the
  Model Service already speaks. The Model Service is the platform call that generates. Its
  adapters translate that shape on the way out to each vendor. A second shape inside Session
  is a second translation layer and a second set of bugs.
* **One user beat, several rows**: User text, then assistant `tool_calls`, then the tool
  result, then assistant prose. "Start a return for order A14" becomes those four rows: the
  ask, `tool_calls[check_order]`, `{status: delivered, opened: true}`, and "Opened items
  have a restocking fee; continue?" Store every row. A hole on the next generate is where
  the model invents a tool result.
* **AddMessages takes the batch**: One write for the whole tool-using turn. A crash must not
  persist the tool call without the tool row.
* **Metadata**: Small fields so "only iOS users lose history" is a query, not a prose hunt.
  Channel, locale, product surface. A fact that must survive truncation is a memory key, not
  a blob on the session row.

## 2. The five calls

Workflows need to start or resume, list a user's threads, append a turn, read history, and
forget. Five RPCs, two groups. Without them, five slightly different REST apps appear.

```
  session lifecycle          messages
  ------------------         --------
  GetOrCreateSession         AddMessages
  ListSessions               GetMessages
  DeleteSession
```

* **GetOrCreateSession**: One call so Sam does not race get-then-insert on the first chat.
  Pass `session_id` to resume. Omit it, or pass null, to start. Return the id every time.
  Clients do not mint ids the service never saw unless you explicitly allow client ids, and
  then those ids are unique and authorization-checked.
* **ListSessions**: How the UI shows "yesterday's return." Summaries only: id, `updated_at`,
  title or first user line. The model does not read 400 full transcripts through this call.
* **AddMessages**: The batched write from section 1. N messages, one transaction.
* **GetMessages**: Two readers, two arguments. `limit` and `before_position` are for a UI
  and for "give me the raw log." `token_budget` and `strategy` are for model context. A
  person debugging a complaint wants bytes. The model wants a budgeted view.
* **DeleteSession**: Retention and "forget this chat." Whether memories cascade is an
  application policy you write down. The call does not guess.
* **Authorization**: A user lists their own sessions. A workflow identity is checked too.
  Guessing a UUID is not enough to call `GetMessages`.

## 3. Storage backends

The contract does not name Postgres. One team's audit log is another team's hot cache.
Forcing one database either overpays (a full VM for a game that can lose a room) or
under-complies (Redis for a clinic).

Pick by the failure you will accept.

1. **Relational** (Postgres, MySQL): Crash-safe rows, SQL, joins, an audit trail,
   point-in-time recovery. Use it for health and money, where you must say who touched a
   row. It breaks when a disposable game room cannot justify the schema, the migrations, and
   the VM.
2. **In-memory** (Redis): Sub-millisecond reads and a short TTL. Use it for an ephemeral
   game master. It breaks on durability and on eviction surprises. A TTL on a shopping chat
   is a feature. A TTL on "enterprise discount" is a bug: that fact wanted user-scoped
   memory, not a 24-hour session row.
3. **Document** (Mongo and similar): Records whose shape can change without a migration. Use
   it when the payload outruns a fixed schema. It breaks when you needed joins and
   crash-safe invariants.
4. **Serverless** (Dynamo and similar): Scale to zero for burst-and-idle. Use it when the
   shop sleeps most of the day. It breaks when your queries do not match the key design, and
   when you cannot leave the vendor.

* **One ABC, many backends**: Apps may use different stores behind `SessionStorage`.
  Register them on the Session Service so the org still has one contract. A teaching
  platform ships Postgres and keeps the ABC real, so a later Redis class is not a fork of
  the servicer.
* **The interface stays the five operations**: Python in, Python out. Postgres speaks SQL.
  Redis speaks commands. Tests speak an in-memory fake. The servicer does not import a
  database driver. Analytics belongs on another store or a replica. A `query_sql` method
  means the abstraction has leaked.

```
  class SessionStorage(ABC):
      get_or_create_session(user_id, session_id=None) -> Session
      list_sessions(user_id) -> list[Session]
      add_messages(session_id, messages) -> None
      get_messages(session_id, ...) -> list[Message]
      delete_session(session_id) -> None
```

### Watch for

* Tests that pass only on Postgres. That is a leak, not an interface.
* A clinic on Redis, or a game room on a regulated relational cluster. You paid for the
  failure you did not name.

## 4. Postgres behind the interface

The teaching backend is Postgres. The habits transfer: one method maps to one storage verb,
writes share a transaction, connections come from a pool.

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
                                   position
```

* **Two tables**: The session row is the handle: resume, list, access control, last
  activity. Messages grow. A JSON array on the session row rewrites a large row on every
  append, turns pagination into an application-side slice, and locks the session during a
  tool turn. Rows scale. The blob surprises you around 5,000 messages.
* **The read index**: `(session_id, position)` or `(session_id, created_at)`. Do not select
  every column of a 10,000-turn thread on each hop. `GetMessages` is where the budget
  starts, before any summary exists.
* **Tool fields stay JSON**: The Model Service already treats them as structure. Leave
  `tool_calls` out of free-text `content` so the next turn can replay the call.
* **get_or_create is one transaction**: A provided `session_id` is selected and authorized.
  A missing id is an insert that returns the new id. A unique constraint beats
  check-then-act. Two first messages with no id must not create two sessions.
* **add_messages is one transaction**: N inserts commit together. You persist the tool call
  and its result, or you persist neither.
* **Pool and statuses**: A connection per RPC will exhaust the database. Map storage errors
  to `NOT_FOUND` and `PERMISSION_DENIED`. Driver exceptions stay inside the storage class.
* **List and delete**: `list_sessions` is `WHERE user_id = $1 ORDER BY updated_at DESC` with
  a cap. `delete_session` removes messages (or trusts `ON DELETE CASCADE`) and then the
  session row. Session-scoped memories join that transaction only when the product flag says
  they cascade.
* **A thin servicer**: The gateway delivers bytes to the servicer. The servicer translates
  and calls one storage method. If `GetMessages(strategy=...)` lives here, it calls helpers.
  Summary SQL does not sit in the protobuf layer. The workflow speaks `platform.sessions`.
  It does not speak SQL. An in-memory fake of the same ABC is how CI stays honest without a
  database.

## 5. The client Sam actually calls

Sam will not write protobuf. An awkward client sends him back to pasting history into the
prompt.

```
  session = platform.sessions.get_or_create(user_id, session_id)
  history, _ = platform.sessions.get_messages(session.id, limit=20)
  platform.sessions.add_messages(session.id, [user_msg, assistant_msg])
```

* **SessionClient**: A `BaseClient` with `service_name="sessions"`. Methods:
  `get_or_create`, `list`, `add_messages`, `get_messages`, `delete`. Python lists in, Python
  objects out. Same lazy channel as `ModelClient`.
* **The id comes back on every path**: The frontend sends it on the next click. Lose it and
  every click forks a new conversation.
* **Storage deadlines**: These calls wait on a database, not on generation. A 30-second
  model timeout is the wrong default for `get_or_create`.

## 6. Model-managed memory

The log is faithful and dumb. Treat the context window as RAM and this store as disk. The
model writes facts it will need after the turn that contained them is evicted. The platform
job is an API, so workflows stop hiding a `memories` blob in session metadata.

```
  memories = platform.sessions.get_memory(user_id)
  recent, _ = platform.sessions.get_messages(session_id, limit=20)
  messages = assemble(system + memories, recent, user_text)
  reply = platform.models.chat(messages, tools=["save_memory", ...])
  # save_memory tool call -> platform.sessions.save_memory(...)
  platform.sessions.add_messages(session_id, turn_including_tools)
```

* **Scope**: Every entry has `user_id` and an optional `session_id`. User scope keeps
  `discount_tier=enterprise` after the chat ends. Session scope keeps "wants a navy case" on
  shopping thread A, where a TTL can be a feature, and off thread B. No owner at all is a
  global brain that fails access control.
* **Delete is two verbs**: Cascade-on-session-delete is a product flag. A clinic leaves user
  memories in place. A retail experiment cascades session-scoped keys. "Forget me" is
  `clear_user_memory`, which clears the user map. Deleting the last session does not do that
  job.
* **MemoryEntry**: A `key` the `save_memory` schema advertises (`discount_tier`,
  `preferred_channel`, `open_ticket_id`), a JSON-able `value`, `user_id`, optional
  `session_id`, `created_at`, `updated_at`. Values update in place. "They are on the
  enterprise plan now" overwrites. A history of overwrites is an audit table with its own
  retention. A catch-all `notes` key is a document store with no query.
* **Four verbs on the same ABC**: `save_memory(user_id, key, value, session_id=None)`,
  `get_memory(user_id, key=None, session_id=None)` for one entry or many,
  `delete_memory(user_id, key)`, `clear_user_memory(user_id)`. In Postgres, uniqueness is
  `(user_id, key)` or `(user_id, session_id, key)`. Write that rule once so the first
  implementer does not invent a private one.
* **JSON on the wire**: `platform.sessions.save_memory` and `get_memory`. Protobuf carries
  bytes or a string. Handlers decode JSON into dicts for Sam. They do not pickle Python
  objects.
* **Load before generate, save on the tool call**: Memories enter the prompt as compact
  system lines, or the model queries them with a tool. The tool-call runtime invokes
  `save_memory`. This service persists the entry.

### Watch for

* The model says "I'll remember you're on enterprise" and never calls `save_memory`.
  Truncation deletes the only copy.
* Saving every sentence. You grow a second transcript and still cannot query a column-shaped
  key.

## 7. The token budget

Stored history is not the same as prompted history. The platform offers strategies. The
workflow picks one and names a budget. A password-reset bot and a clinic bot do not share a
hidden `[-20:]`.

```
  | system | tools | data chunks | memories | HISTORY | output reserve |
  |  2k    |  3k   |     4k      |    1k    |   ???   |      2k        |
```

* **The remainder is history**: A 4k persona, 40 tool schemas, retrieved chunks, and a
  memory dump all eat the window before the transcript does. People hear "128k" and still
  expect 200 verbatim turns. On a 32k window the slices above already spend 12k, so history
  and the reply share what is left.
* **Count with the model's tokenizer family**: A different tokenizer makes "return policy"
  two tokens in one service and four in another. Twelve workflows then disagree about the
  same budget.
* **Who owns the slice**: The Session Service assembles HISTORY, and often the memory lines.
  Other platform services own their own slices. A fight with no numbers ends as a context
  error blamed on the vendor.
* **The output reserve**: Leave room for the completion. If history consumes whatever is
  left, the reply is cut mid-sentence or the call fails.

## 8. Truncation and summarization

Fast and lossy, or slower and still lossy. Name which one you are running.

1. **Truncation**: Walk backward from the latest message. Keep rows while tokens remain.
   Drop the prefix. Use it for independent FAQs, where turn 1 does not constrain turn 40. It
   breaks when the constraint is in the prefix: kept `[m31..m50]`, dropped `[m1..m30]`, and
   `m1` was "our enterprise discount." Always keep a leading system message if it is in the
   list. Do not split a tool-call group and leave a call without its result. The token count
   lives in the service, or in one shared tokenizer library.
2. **Summarization**: Compress the old slice with a model call. Keep recent turns verbatim.
   The block is labeled, for example
   `SUMMARY of m1–m30: enterprise discount; opened laptop; wants label`, then `[m31..m50]`
   verbatim. Use it when a narrative fact in the prefix still matters and a short paragraph
   is acceptable. It breaks when the paragraph softens "enterprise" into "some discount": a
   false memory, and a billing incident in a helpful tone. Mark the block as a summary so
   the model does not quote it. Cache it by the range of message ids. Do not re-summarize an
   idle thread on every click. Use a cheap model for the hop so compression does not cost
   what the user-facing reply costs. Session calls the Model Service for that compression.
   Session does not generate the reply the user sees.

* **Keys outrank paragraphs**: Put `discount_tier` in a MemoryEntry as well. The key
  survives after the summary loses the word.

### Watch for

* Dropping the system message while trimming user turns. That is a different bug from a
  short history.
* A summary cache that ignores the message-id range. The next click pays for a new summary
  or serves a stale one.

## 9. Hierarchical memory and other sessions

One strategy is a blunt instrument. Long chats want layers. Other days want search.

```
  TIER 1  model-managed memories    structured, must not drop
  TIER 2  rolling summary           older narrative, lossy
  TIER 3  recent verbatim           last N turns, full nuance
```

* **Hierarchical assembly**: Memories, then the rolling summary, then recent verbatim, plus
  document chunks from the library when the workflow has them. If the pie is still too
  small, shrink tier 3 first, then re-summarize tier 2, and leave tier 1 intact. Offer this
  as `GetMessages(strategy="hierarchical", token_budget=15000)`.
* **Retrieval over past sessions**: "What did you recommend for headaches last time?" can
  sit in another session from months ago. Tiers inside this thread cannot see it. Search
  user-scoped indexes of session text and insert hits labeled as prior conversation, so the
  model does not treat them as this turn's user voice. A miss looks like forgetting. The
  persona should say to ask when there is no hit, and the window should be allowed to admit
  it does not hold a life history. The index and the call exist so each team does not embed
  its own search stack. `search(query, filters={user_id, source: sessions})`.

| Conversation | What you run |
|---|---|
| Independent FAQs | `strategy="truncate"`, for example `limit=20` |
| One-session shopping | Hierarchical, no cross-session search |
| Clinic or long support | Memories, hierarchical, and past-session search |
| Coding thread | Past solutions retrieved, recent turns weighted |

* **Three questions pick the row**: Do users cite earlier turns in this chat? Add hierarchy
  or a memory key. Is there a fact that must never fall out? Memory keys. Do they cite other
  days? Search past sessions. Trigger that search when the utterance sounds like "last
  time," and measure the misses.
* **The workflow still composes**: The platform's job is to stop every team from shipping a
  private `[-20:]`. Published manuals and policies belong in the organizational document
  library. A sentence Maria said in chat belongs here, as a message or a key. Semantic,
  episodic, and procedural memory are research names. You can later store those kinds as
  keys or extra stores. This contract is messages, entries, and a budget.

## Check yourself

1. A user turn with one tool call becomes how many messages? What breaks on the next
   generate if you persist only the final assistant prose?
2. Why do Session messages use the same role and content shape as the Model Service? Name a
   conversion bug that second shape would add.
3. What race does GetOrCreate close that get-then-insert loses when there is no transaction
   and no unique constraint?
4. For a clinic, a game, and an idle-then-spike shop, name a storage class and the failure
   you are accepting.
5. Should deleting a session delete memories? Name one product that cascades and one that
   does not. Which verb still has to exist for "forget this user"?
6. On a 32k window with the example pie (system, tools, chunks, memories, output reserve),
   who owns history, and what happens if that slice has no reserve for the completion?
7. Truncation dropped "enterprise discount" from turn 1. Which mechanisms in this chapter
   would have kept it, and how do a softened summary, a retrieval miss, and a fact that was
   never saved fail differently?
8. Which mechanism answers "as I said twenty turns ago in this chat," and which answers
   "last spring's visit"?
9. Why does the summary hop call the Model Service with a cheap model instead of the
   frontier model that writes the user-facing reply?
10. Name one fact that belongs in a MemoryEntry key and one that belongs only in the
    transcript. What goes wrong if you swap them?
