# 6. Memory and knowledge RAG

Companion notes for **Chapter 6** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

## The mental model

Two facts fight each other. External stores can grow without bound. A
single model call cannot: it has a **context window**. Retrieval is the
bridge — search the large store, seat only the useful slice in this
prompt.

```
  UNBOUNDED (disk, DB, MCP)              BOUNDED (this call)
  -------------------------              -------------------
  wiki / PDFs / tickets                  system + persona
  vector index                           tool schemas
  keyword / BM25 index     retrieval     retrieved slices
  graph / SQL              ---------->   short-term chat
  prior sessions / MCP                   working notes
                                         (then the model runs)
```

The one sentence to remember a year from now: **knowledge is the
library; memory is the lived record; both reach the model only through
retrieval into a finite window.**

Two consequences fall out of that diagram. First, "we added RAG" is not
a product. RAG is an ingestion path plus a query path plus a policy for
what the model is allowed to say when retrieval is thin. Second,
"the agent remembers" is not a bigger prompt. It is a write path, a
read path, and a forgetting path — usually over more than one store.

```
  KNOWLEDGE (library)                 MEMORY (lived)
  -------------------                 --------------
  manuals, policies, catalogs         this session's turns
  relatively slow-changing            last session's facts
  ingested as chunks + metadata       preferences, episodes
  "what is true in the corpus"        "what happened with us"
```

Same primitives (embed, search, fetch). Different freshness, different
write cadence, different hygiene. Mixing the words is how a chat log
gets treated like a policy source, and a PDF gets treated like a
user preference.

## Retrieval in AI applications

The model is asked about a private or recent fact and confidently
invents one when that fact lives only in the weights' imagination.
Keep the fact *outside* the weights. At call time, fetch a small
relevant set and put it in the prompt (or in a tool result the prompt
can see).

Retrieval is not a vendor feature. It is the mechanism that turns
"arbitrarily large external state" into "tokens this call can use."
Unstructured stores hold prose: tickets, transcripts, notes,
preferences written as sentences. Structured stores hold rows and
files you would query with SQL, a filesystem, or an API. Agents use
both. The structured path is often a **tool**. The unstructured path
is often **embed and search**. Production systems use both on the same
question.

What is *not* retrieval: stuffing the last 200 turns into the window
and hoping. That is a context policy. It works until it does not, and
it does not scale to "everything this user ever said" or "the whole
handbook." How a platform truncates, summarizes, and budgets tokens for
the live transcript is a separate service concern. Here, the
agent-shaped question is: **which tool do I call, with which query, and
what do I do with an empty hit list?**

### What can live outside the model

Keep a short inventory so design reviews stop saying "put it in the
prompt":

- **Documents** — handbooks, specs, contracts. Knowledge.
- **Operational records** — error catalogs, SKUs, runbooks. Knowledge
  with nasty exact tokens (codes, IDs).
- **Conversation** — this thread (short-term, usually in the window)
  and prior threads (long-term, retrieved). Memory.
- **Experiential traces** — "last time we refunded this waybill we
  needed a manager code." Memory that behaves like a procedure.
- **User facts** — diet, timezone, account tier. Memory that should
  not be re-asked every session.

If you cannot say which bucket a datum is in, you cannot choose a
store, a retention rule, or a retrieval tool.

### The context window is the real budget

Modern windows are large. They are not infinite, they are not free,
and they are not a substitute for an index. A million tokens of
handbook still has a ranking problem: the model attends unevenly, you
pay for every token, and a contradictory paragraph on page 400 still
gets a vote. Retrieval exists to **choose**, not to **hoard**.

Treat the window as RAM and the index as disk. You would not load the
disk into RAM on every function call. Do not load the corpus into
every agent turn.

## The basics of RAG

RAG (retrieval-augmented generation) is a two-phase machine. Teams
demo the second phase and forget they shipped a data pipeline.

```
  INGESTION (offline / async)            RETRIEVAL (per question)
  ---------------------------            ------------------------
  load docs                              embed the query
  split into chunks                      search nearest chunks
  attach metadata                        (optional: keyword too)
  embed each chunk                       stuff hits into prompt
  upsert into a store                    generate + (later) ground
```

Two models, two jobs. The **embedding model** maps text to a dense
vector so similar meanings sit near each other. The **generator**
(your chat / agent model) writes the answer. They are not
interchangeable. Changing the embedder without re-indexing is how you
search in a space the documents are not in.

A demo that indexes one PDF, asks a question that appears verbatim in
chunk 4, and declares RAG "done" has only exercised half the machine.
Draw both phases. Name the chunker, the embedder, the store, the `k`,
the prompt rule for "no hits," and the grounding rule for "hits exist
but the claim is not in them." Without grounding, retrieval is a
suggestion the model is free to ignore.

### Ingestion is a product decision

Chunking is not "split every 500 characters." It is a bet about the
smallest unit that still answers a question:

- **Too small** — a table cell without its header; a "yes" without
  the policy it answers.
- **Too large** — five topics in one vector; the search hits the
  blob, the generator drowns.
- **No overlap** — a sentence that spanned the cut is in neither
  neighbor's embedding.
- **No metadata** — you cannot filter by product, date, or tenant,
  so every query searches the whole world.

Overlap, heading-aware splits, and metadata (`source`, `as_of`,
`acl`) are the unglamorous half of RAG. An org-wide ingestion pipeline
turns this into a shared service; an agent still inherits whatever you
ingested. Garbage chunks in, confident nonsense out.

### Retrieval is a tool, not only a preprocessor

Classic RAG is a pipeline: query → retrieve → generate, once. An
**agent** can call search as a tool, read the observation,
**reformulate the query**, search again, or switch from vector to
keyword. That is the difference between a chatbot with a vector
appendix and a RAG *agent*. It is also how you get runaway retrieval
loops. Budget the tool. For this chapter, remember: **search/relevance
first**, then wrap it in an agent, then add a second search style.

## Semantic search and document indexing

Indexing transforms a document so a later query can find it. How you
will query is part of how you should index. Phrase match, keyword
match, and meaning match are different indexes even when they sit in
one product.

**Semantic search** ranks by meaning. You do not maintain a synonym
list so that "bike" finds "cycle." The embedding space is supposed to
put them near each other. That power is why RAG felt like magic in
demos. It is also why it fails in production in boring, predictable
ways.

### Three pitfalls that are not bugs in the database

1. **Similarity is not truth.** The nearest chunk to "best treatment
   for a seized lock" may be a blog post that *sounds* like a
   treatment and is wrong, expired, or from a tenant you should not
   search. The index has no notion of correct. It has a notion of
   nearby.
2. **Top-k is a cliff.** If `k=5`, the relevant paragraph at rank 6
   does not exist for the model. Multi-aspect questions (hours *and*
   fees *and* exceptions) need either a larger `k`, query
   decomposition, or more than one search.
3. **The query language is not the document language.** Users say
   "the thing that beeps when the dock is full." The manual says
   "overcapacity alarm (SPK-441)." Pure semantics sometimes bridges
   that. Exact codes often do not. Plan for both.

Stakeholders hear "semantic" and disable keyword search as "old."
Keep a lexical path for identifiers, error codes, names, and jargon
that embeddings smear. Hybrid search is how you stop missing
`SPK-441`.

### Indexing as a contract

Write the contract down before you pick a store:

- What is a document? (file, page, ticket, message)
- What is a chunk? (tokens, heading path, overlap)
- What metadata is filterable?
- Which embedder, which dimension, which similarity?
- When do we re-embed? (doc edit, embedder change, both)
- Who is allowed to hit this index?

If two teams share an index without that contract, you do not have
knowledge. You have a pile. Isolated indexes per tenant or corpus are
an infrastructure concern; the agent must still pass the right
collection name or tool. Searching the wrong index is a silent
security bug.

## Vector similarity

Before dense embeddings, it helps to remember a method that does
**not** capture meaning, because you will still want it.

### Lexical vectors (TF-IDF and friends)

TF-IDF (and BM25 in the ranking world) turn a text into a **sparse**
vector: most dimensions are words that are absent. A term scores high
when it is frequent *here* and rare *across the corpus*. That is a
statement about distinctive words, not about concepts.

```
  query: "vehicles"
  doc:   "the cars in bay 3"
  TF-IDF: miss  (no shared term)
  dense:  hit   (nearby meanings, if the embedder learned them)
```

The reverse failure is just as real:

```
  query: "SPK-441"
  doc:   "SPK-441  dock overcapacity alarm"
  TF-IDF / BM25: hit  (exact token)
  dense:           maybe miss, maybe bury under "alarm" essays
```

Treating TF-IDF as a broken embedding model misses the point. Treat it
as a **precision instrument for tokens**. It is fast, inspectable, and
often wins on SKUs, names, and codes. Semantic embeddings win on
paraphrase. Hybrid systems run both.

### Dense embeddings

An embedding network is trained so that texts people would call
"about the same thing" land near each other in a high-dimensional
space. You do not write the features. You pick a model, you accept
its biases, and you **must use the same model** (or a documented
compatible one) at query time that you used at index time.

Workshop instincts that save weeks:

- Do not mix embedders in one collection.
- Do not assume "OpenAI embedding" means one vector size forever;
  pin the model name in the index metadata.
- Short queries and long chunks live in the same space only because
  the trainer said so; very short queries ("hours?") are weak. Expand
  them (HyDE, rewrite, or a keyword fallback).
- Multilingual and code-heavy corpora need embedders that actually
  saw those domains.

### Cosine similarity and distance

Most stacks compare two vectors by **cosine similarity**: how aligned
their directions are, ignoring length. Score near 1 means "same
direction," near 0 means "unrelated under this embedder," near -1
means "opposite" (rare in well-trained text spaces, but the math
allows it).

Some stores return **distance** instead of similarity. A common
conversion:

```
  cosine_distance = 1 - cosine_similarity
```

Under that convention, **0 is nearest**, not best-friend-high-score.
Read the docs for *your* store before you sort. Ranking the wrong way
is a one-line bug that looks like "embeddings don't work."

Chroma's examples in this chapter's spirit often surface **distance**.
If you port the idea to another store, check whether `0.12` is good
or terrible.

A tiny in-memory toy (sparse vectors, cosine, top-n) is worth
building once so the later database feels like a persistence layer,
not magic. Do not ship the toy. Ship a store that handles upserts,
filters, and persistence.

## Vector databases

After you have vectors, you need somewhere to put them that can
answer "nearest to this query vector" quickly, with metadata filters,
and without re-embedding the corpus on every process start.

"We need Pinecone" as the first architecture sentence skips the real
work. Name the **interface**: add embeddings, query by vector (and
optionally by text, if the store embeds for you), filter on metadata,
persist. Then pick an implementation that matches deployment, not a
blog post.

### Chroma as an example, not a religion

Chroma is a reasonable **local** vector store for workshops and small
projects: a client, a collection, `add`, `query`. It makes the
lifecycle visible. It is not the only choice and not automatically
the production choice.

Other shapes you will meet (so you do not think the API *is* RAG):

- **Embedded / local** — Chroma, LanceDB, a SQLite extension. Great
  for a single agent process.
- **Postgres with a vector type** — one database you already run;
  pgvector is a common direction.
- **Dedicated services** — Qdrant, Weaviate, Pinecone, etc. Ops,
  SLAs, and hybrid features vary.

This folder does not pick your vendor. It asks whether the *agent*
can call `search` without knowing which of those is behind the tool.

### Querying a collection

The loop, regardless of brand:

1. Embed the query with the **same** embedder used at ingest (unless
   the store does it and you configured that embedder there).
2. Ask for top-`n` nearest neighbors, optionally filtered
   (`source = "spoke-ops"`, `as_of >= ...`).
3. Read **documents + distances/scores + metadata + ids**.
4. Decide: enough? reformulate? keyword fallback? refuse?

Empty results are a first-class outcome. The persona should say "I
don't have that in the handbook" rather than free-associate. That
sentence is a product requirement, not a nicety.

Ids matter. You need them to de-duplicate, to cite, and to delete
when a document is withdrawn. If your only handle is "the third
chunk we got back," you cannot run a memory hygiene job.

### What the store does not do

A vector DB will not:

- verify the chunk is true,
- keep the generator honest,
- chunk your PDFs,
- handle ACLs unless you put ACL fields in metadata and **filter**,
- re-index when legal withdraws a paragraph, unless you build that
  job.

Those are your ingestion, policy, and evaluation layers. Do not
blame cosine distance for a missing `delete`.

## Practical RAG agents

Similarity search is a foundation. It is not a sufficient retrieval
policy. Practical systems start from **search and relevance**, then
wrap an agent around a vector tool, then add a lexical tool and a
way to merge lists.

### Everything begins with search and relevance

Feeding the generator twelve "semantically close" chunks and hoping it
sorts truth from rhyme is a retrieval policy failure dressed as a
prompt problem. Catalog the failure modes *before* you write the
agent. Each row wants a retrieval move, not a longer system prompt.

| Failure | What it looks like | Move |
|---|---|---|
| Exact token miss | Query `SPK-441`, hits are essays on "alarms" | Keyword / BM25, or hybrid |
| Jargon mismatch | User says "beep dock"; manual says "overcapacity" | Semantic *or* a synonym list *or* hybrid |
| Near-duplicates | Twelve copies of the same spec sheet | Dedup by id/hash; MMR-style diversity |
| Stale chunk | Policy changed; old paragraph still nearest | Metadata `as_of`; re-ingest; time filter |
| Wrong corpus | HR index answers an ops question | Isolated collections; tool routing |
| Rank cliff | Answer sat at position 8, `k=5` | Raise `k`; split the question; second hop |
| Thin hits | Store has nothing | Refuse; do not improvise policy |

Vector search is the right hammer for "say it another way." It is
the wrong hammer for "match this identifier" and a sloppy hammer
for "the ten near-clones all look relevant."

Relevance is not only nearest-neighbor. It is **the set of chunks
that would let a careful reader answer**, with citations, without
contradiction. If you cannot explain why a chunk is in the prompt,
it should not be in the prompt.

### A vector-search RAG agent

Build the simple one first so the hybrid one has a baseline.

Shape:

1. Load a corpus (workshop: a single long text — ops manual, script,
   catalog).
2. Chunk by token budget with a little overlap.
3. Embed; upsert into a local collection (Chroma in the book's
   lab; any store that honors the interface).
4. Expose **one tool**, e.g. `search_handbook(query) -> passages`.
5. Persona: you answer from passages; if they do not contain it,
   say so. Cite chunk ids if you can.

That is already an agent: sense (user question), plan (form a
search query), act (tool), learn (read hits, maybe search again).
The anti-pattern is stuffing "always call search" into a 2,000-word
prompt. Prefer a **clear tool name and docstring**; let the model
choose, then measure whether it does.

Grounding belongs in the instructions as a *constraint* ("only
claims supported by hits") and later as a *separate checker*. Do
not confuse the two. A constrained persona still hallucinates. A
grounding agent or guardrail is how you catch it.

Workshop build order that keeps you honest:

- Empty collection → tool returns "no hits" → agent must refuse.
- One obviously answerable fact → agent must cite it.
- A near-miss paraphrase → vector path should still find it.
- An exact code → note the miss; that ticket files the hybrid
  section.

If you skip the empty-collection case, you will never see whether
"I don't know" is actually implemented.

### A hybrid-search RAG agent

Run **keyword** and **vector** on the same query. Each returns a
ranked list. The scores are not comparable: BM25 and cosine live
on different planets. You need a **fusion** step.

**Reciprocal Rank Fusion (RRF)** ignores raw scores and uses rank
position. A typical fused score for a document is the sum over
lists of `1 / (k_rrf + rank)` with a small constant `k_rrf` (60 is
a common default). Items that appear high in *both* lists rise.
Items that win only one list still appear, which is the point:
the error code hit is not vetoed because its embedding was mediocre.

```
  query
    |-- keyword list  (SPK-441 at rank 1)
    |-- vector list   ("dock full alarm" essay at rank 1)
              |
              v
           RRF merge
              |
              v
        shared top chunks --> generator
```

Two ways to give this to an agent:

- **One hybrid tool** — the store or your wrapper fuses internally.
  Simple for the model; you tune fusion offline.
- **Two tools** — `search_lexical` and `search_semantic`. The agent
  chooses, or calls both and synthesizes. More agency, more ways to
  skip the useful one, more traces to inspect.

If a shared search endpoint already fuses, the agent's tool should
call *that* rather than reimplement RRF in the persona.

Fusion constants cargo-culted from a blog and never measured will
quietly miss one class of questions. Hold out questions that *need*
exact tokens and questions that *need* paraphrase. Plot hit rate at
rank 5 for vector-only, keyword-only, and fused. Change one knob.

Agent-directed chaining is a third hybrid: first keyword, if empty
then vector, or the reverse. That is a policy you can test. It is
not automatically better than RRF. It is more interpretable.

## Memory with MCP

RAG-for-documents and RAG-for-memory share retrieval. They do not
share **writes**, **identity**, or **forgetting**. MCP is how you
attach stores as tools without baking Chroma or a graph client into
every agent binary. Community servers exist for graphs, vectors, and
files. Use them as *sockets*, not as an excuse to skip the memory
policy.

### Memory form versus agent function

Cognitive vocabulary (short-term, long-term, episodic, semantic,
procedural) is a **filing system for engineers**, not a claim that
the agent has a hippocampus. If the words get in the way, drop
them and name the parts you actually run:

| Part | Job |
|---|---|
| Context window | Tokens in *this* call (working / short-term) |
| External store | Survives the process (long-term) |
| Scratch / state | Plan, notes, tool traces during a task |
| Retrieval | Query that copies a slice into the window |
| Write policy | What gets stored, under which id, with whose consent |
| Forgetting | TTL, summarization, deletion, contradiction |

"Add memory" often becomes a second vector index of raw chat, queried
the same way as the employee handbook. Split **knowledge writes**
(ingest a doc) from **memory writes** (record that this user is
vegetarian, or that last Tuesday's refund needed a manager). Different
schemas, different tools, different retention.

What agents need is mundane: persist across sessions, retrieve
what is relevant, splice it into the prompt, and not retrieve
what is expired or someone else's. The rest is implementation.

Short-term memory is usually the conversation buffer. It dies
when you truncate. Long-term memory is a store you query on
purpose. If you never write, you do not have long-term memory.
If you write everything, you have a landfill.

### Graph memory over MCP

Vector search is good at "passages like this." It is weak at
"who owns the autoclave, and what broke the last time we used
it?" That is relational structure: entities, types, edges,
observations.

```
  (Maya:Person) --works_on--> (Kite:Project)
  (Kite:Project) --uses--> (Autoclave-3:Equipment)
  (Autoclave-3) --observation--> "seal failed 2026-08-12"
```

A **graph store** lets the agent ask by entity, by type, or by
relation, then pull neighboring facts. Graph RAG is this idea
applied to a corpus: extract entities while ingesting, retrieve a
subgraph, then generate. For *memory*, the same shape records
people, constraints, and events from conversation.

MCP is a practical way to attach such a store: tools like
"search nodes," "add observation," "find related." A sequential-
thinking scratchpad stores *plans for the current job*. A graph
server stores *world state that should still be true tomorrow*. Do
not use one server for both jobs without naming the difference.

Extracting a graph with an LLM on every message and never merging
duplicate nodes ("Maya", "maya@lab", "Dr. Chen") is how graphs rot.
Identity rules. Stable ids. Observations attached to nodes, not new
nodes per synonym. Periodic cleanup. Graphs rot faster than vector
piles because bad edges look like knowledge.

When a graph helps:

- multi-hop questions ("equipment on Maya's project that failed"),
- constraints that are relations ("must not email legal@ without
  cc'ing Maya"),
- inventories and org charts.

When it is theater:

- a single FAQ corpus with no entities worth naming,
- a team that will not maintain extraction quality.

### Hybrid memory

Hybrid *search* fused two rankings over **documents**. Hybrid
*memory* gives the agent **two write/read tools** over different
shapes: a semantic/vector store for prose ("what we talked about")
and a graph (or SQL) for facts and relations ("who / what / bound
to whom").

The orchestration can live in **instructions**: on each turn,
search semantic memory, search the graph, synthesize, then decide
what to capture from this turn. That is easy to demo and easy to
drift. Prefer:

- explicit tools with boring names,
- a fixed order you can trace (search before answer; write after
  you know what was true),
- a cap on how many memories you inject.

```
  turn starts
    --> semantic search (conversation / notes)
    --> graph search (entities / constraints)
    --> synthesize into a short "remembered" block
    --> answer using that block + live tools
    --> write back: new fact? new episode? skip?
```

MCP shines here because each store is a server. The agent binary
does not grow a new client library per store. You still own the
**policy**: what is allowed to be written (PII, secrets), when
writes happen (every turn is usually wrong), and how conflicts
resolve (user says "I eat fish now").

A mandatory theatrical prefix ("Remembering...") as a substitute for
a real workflow can be skipped by the model. If you need a ritual,
make it a tool call you can see in a trace, not a string the model
can skip. Later evaluation can score whether the search happened.

### Semantic, episodic, and procedural

Psychologists split long-term memory by *what* is stored. Steal
the split for schemas, then augment each row with a semantic
handle so vector search can find it.

- **Semantic** — durable facts and meanings: "Cedar Ridge closes
  Mondays," "user prefers aisle." Vector or key-value. Invalidate
  on contradiction.
- **Episodic** — events: "on 12 Aug the user asked to move the
  reservation after the strike notice." Relational (timestamp,
  actors, outcome) plus a prose summary you can embed.
- **Procedural** — how to do a thing this agent already figured
  out: "to refund a Spoke day-pass, pull the trip id, then call
  `credit_wallet`." Steps, preconditions, last-success. SQL or
  a playbook table; embed the description so a future query
  finds the playbook.

**Semantic augmentation** (workshop meaning): take an episode or
procedure and ask a model to write the *questions that should
retrieve it later*. Store those questions (or their embeddings)
next to the record. Retrieval then matches how users actually
ask, not how the log was worded.

```
  episode: "strike, moved reservation, waived fee"
       |
       v
  LLM proposes trigger questions
       |
       v
  embed questions + summary --> vector side
  keep structured fields    --> relational / graph side
```

Do not augment by inventing facts. Augment by inventing **queries**.
The record stays the source of truth.

A row that is only a vector is hard to update ("delete the fee
waiver"). A row that is only SQL is hard to find with a vague
user utterance. Hybrid rows cost more design and less regret.

### Compression and forgetting

Stores clog. Duplicates, stale prefs, every "ok" in a chat.
Biological metaphors are optional; the engineering is not:
**compress** what you keep, **forget** what you should not
retrieve.

Compression (same family as augmentation, extra step): cluster
similar memories (embeddings + k-means or "same entity id"),
then summarize the cluster into one record. Keep a pointer to
the members if you still need audit. This is how a month of
scheduling chatter becomes "user usually cannot do Tuesdays."

```
  many near-duplicate memories
           |
        cluster
           |
        summarize  -->  one compact memory
           |
        (optional) drop or archive members
```

Forgetting is a policy, not a crash:

- **TTL** — session scratch dies in hours; dietary facts do not.
- **Frequency** — unused procedural tips decay; daily-used ones
  stay.
- **Supersede** — new fact replaces old; do not retrieve both.
- **User deletion** — a legal requirement, not an afterthought.
- **Never-store** — secrets, raw card numbers, other people's
  data that landed in a paste.

Summaries become false memories ("user is vegetarian" when they
said "vegetarian this trip") when the compact record loses
provenance. Keep the source episode id. Prefer replace-with-
structured-field over lossy prose when the field is operational.
Summaries are for *search and context*, not for *authorization*.

Truncation and hierarchical session summaries for the live
conversation buffer are a token-budget algorithm for the window.
This chapter's forgetting is the **long-term store** growing mold.
Different jobs.

## Check yourself

1. A teammate says "the model has a million-token window, so we
   don't need RAG." Which two problems does a window *not* solve
   even if the whole corpus fits?
2. Draw ingestion vs retrieval for a handbook that changes weekly.
   Where does `as_of` metadata get written, and where is it
   filtered? What happens if you only do one of those?
3. Why are the embedding model and the generator two jobs? Give a
   failure that appears if you upgrade one and not the other.
4. Query `SPK-441` vs query "the dock that won't accept more
   bikes." Which retrieval family do you want first for each, and
   what does hybrid+RRF buy you that "pick one" does not?
5. A store returns cosine *distance*. You sort descending and
   keep the top 5. What did you just retrieve, in plain language?
6. Knowledge vs memory: the user says "I'm vegetarian." The
   cafeteria PDF lists Monday menus. Which store gets the write,
   which gets the read on "what can I eat with you today?", and
   what goes wrong if both are the same collection with no types?
7. Graph vs vector: "who owns autoclave 3 and what failed last?"
   Why might a nearest-neighbor paragraph search be a worse
   interface than a neighborhood query? When would the graph be
   wasted work?
8. Name the four mechanical pieces (window, store, scratch,
   retrieval) for an agent that must remember a preference
   *across* weekends. Which piece is MCP replacing?
9. Semantic augmentation of an episode: what is allowed to be
   model-generated (the triggers) vs what must stay literal
   (the record)? Why?
10. Compression produced "user hates Tuesdays" from three chats
    about *this month's* strike. What forgetting or provenance
    rule would have stopped that compact memory from living
    forever?
