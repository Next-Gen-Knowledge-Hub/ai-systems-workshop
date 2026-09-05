# 5. The Data Service

Companion notes for **Chapter 5** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is how the platform turns files into **searchable
organizational knowledge**. Sessions remember what Maria said yesterday.
Data remembers what Legal published last Tuesday. Skip this chapter and
every workflow either stuffs a PDF into the prompt (cost, noise, stale
pages) or each team ships a private chunker (sprawl with extra vectors).

The Agents track is a different book. Agent-shaped RAG — retrieve as a
tool, reformulate the query, maybe retrieve again — lives in
[agents ch. 6](../../agents/6-memory-and-rag/). This folder stays on the
**Data Service**: isolated indexes, one ingestion pipeline, hybrid search
the organization can operate. Mention that agent chapter. Do not rewrite it.

## The mental model

```
  PDF  DOCX  HTML  MD  TXT          caller metadata
       \  |  |  |  /                      |
        \ |  |  | /                       v
     detect -> extract -> sections + auto fields
                              |
                              v
                         chunk (index config)
                              |
                              v
                    embed via Model Service
                              |
                              v
           +-------- isolated INDEX (one team) --------+
           |  chunk text  |  vector  |  keyword tokens |
           |  document id |  JSONB metadata            |
           +-------------------------------------------+
                         |              |
                search() (cosine)   keyword_search()
                         |              |
                         +------+-------+
                                v
                         RRF merge (hybrid)
                                v
                    ranked chunks -> workflow context
```

The one sentence to remember: **an index is a product boundary**, not a
table. Embedding model, chunker, and corpus are frozen together so a
support query cannot "helpfully" retrieve a sealed legal memo, and so
vectors from two models never share a similarity operator.

Chapter 1 called this half of **context-aware intelligence**.
[Session Service](../4-session-service/) is conversational state. This
service is the library. Mixing the two is how "as I mentioned yesterday,
our enterprise discount" disappears into a PDF corpus that never heard
Maria speak.

Trade-offs for vector-only vs keyword-only vs hybrid, and for isolated
indexes vs one mega-corpus, are on one screen in
[`TRADEOFFS.md`](../../TRADEOFFS.md) (RAG table). Use this folder for the
*service* that implements those rows.

## From documents to searchable knowledge

Start from a fact, not a framework. Somewhere in a 40-page returns PDF,
page 3 says electronics can come back within 30 days with receipt and
original packaging, with a restocking fee if opened. A customer asks,
"What happens if I return a laptop I already opened?"

Keyword matching looks for *laptop*, *opened*, *return*. The paragraph
might say *electronics* and *original packaging*. Language flexes. Exact
tokens miss. The move that survives paraphrase is **semantic search**:
compare *meanings*, usually as dense vectors, at the grain of a *passage*,
not a whole manual.

You cannot compare a ten-word question to a 50-page blob in any useful
way. The blob mixes return windows, warranty disclaimers, and shipping
tables. The comparison has to happen on pieces that are small enough to
be about one idea and large enough to still be that idea. That is the
whole pipeline: extract text, keep structure, attach metadata, chunk,
embed, store, retrieve.

Naive alternatives fail in production for boring reasons:

- **Paste the PDF into the prompt.** Hits the window, bills tokens,
  still misses the right paragraph in the noise, and updates require a
  deploy or a prompt edit.
- **One regex over raw bytes.** Breaks on columns, headers, and "laptop"
  vs "electronics."
- **Each squad's notebook chunker.** Four embedding models, four notions
  of "document id," four ways to forget a retracted policy.

The Data Service exists so a workflow author calls `ingest` and `search`
(or `hybrid_search`) and does not become a PDF engineer. Parsing, chunk
overlap, and vector backends are **platform work**. Choosing *which
index* and *which filters* is **product work**.

Write the gap for *your* corpus, not the book's laptop. If you cannot
name the passage that should answer a real user question, you are not
ready to tune `top_k`. You are ready to admit the library is not
searchable yet.

## Indexes: organizing knowledge

Before a file is parsed you need a unit that groups related knowledge and
keeps unrelated knowledge apart. In this service that unit is an
**index**: a named, isolated collection with its own embedding
configuration and search behavior.

Sessions in chapter 4 each own a transcript. Indexes each own a corpus.
A team might keep product docs in one index, HR policy in another,
runbooks in a third. The names are cheap. The isolation is the feature.

### Why isolation matters

A platform that serves many teams will otherwise search the union of
everything. Support maintains troubleshooting guides. Legal maintains
compliance PDFs. "How to reset a device" must not surface a data-
retention clause. "Data retention requirements" must not surface a
consumer factory-reset wiki.

```
  Support workflow                         Legal workflow
         |                                        |
         v                                        v
  search(index=support_kb)              search(index=legal_kb)
         |                                        |
         v                                        v
  +------------------+                 +------------------+
  | embed: small     |                 | embed: legal-ft  |
  | chunk: recursive |                 | chunk: structure |
  | docs: runbooks   |                 | docs: statutes   |
  +------------------+                 +------------------+
         ^                                        ^
         |         Data Service (one API)         |
         +------------------+---------------------+
```

Isolation is not a `WHERE team =` filter you remember to add. Filters
fail open when someone omits them. **Scoped search** means the vector
space itself is per index. A support query never ranks legal chunks
because those rows are not in the candidate set.

Isolation also lets configuration diverge. Dense regulatory prose may
want an embedding model trained nearer that domain. API docs may want a
smaller, cheaper model and tighter chunks. Chunk size that is right for
a FAQ is wrong for a statute. Independent knobs are why you do not run
one global "company brain" with a single `chunk_size=512`.

Security reviews should ask: *which indexes can this workflow name?* If
the answer is "whatever string the prompt interpolates," you have a
retrieval confused-deputy, not a library.

### Index configuration

Create-time config is the contract between a team and the retriever.
Minimally you need:

- **Name** — stable identity (`support.troubleshooting`, not `kb2`).
- **Embedding model and dimensions** — how text becomes a vector.
- **Chunking strategy and sizes** — how text becomes passages.
- **Optional metadata schema** — which filter keys are expected.

The embedding fields are **immutable after create**. Vectors from
different models live in different spaces. Mixing them in one index
silently corrupts nearest-neighbor. The platform should reject "please
switch this index to `text-embedding-3-large`." The escape hatch is a
**new index**, re-ingest, cut search over, delete the old one. That is
annoying on purpose. Annoying is cheaper than undiagnosable 0.72
similarities.

Chunking parameters belong on the index for the same reason: every
document in the collection must be split with the same rules or you
cannot reason about recall. Changing overlap later without re-ingest
means half the corpus is the old grain.

Treat metadata schema as a mild type system for filters. If every
runbook should carry `audience=customer|internal`, say so at create
time so ingest can fail loud when a caller omits it.

### Index operations

Lifecycle is unglamorous and you will miss it the week Legal asks "what
do we actually have?"

- **Create** — allocate the named collection and freeze config.
- **Get** — config plus stats: document count, chunk count, created_at,
  last_ingested_at.
- **List** — what the *caller* may see, not a dump of every tenant.
- **Delete** — drop documents, chunks, vectors, keyword postings. This
  is a data-destruction API. Gate it like one.

`last_ingested_at` is an ops signal. An index that has not ingested in
90 days is how "stale knowledge" from [chapter 1](../1-why-a-platform/)
shows up as a timestamp instead of a customer complaint.

Owner is not decoration. It is who gets paged when embeddings start
failing, and who is allowed to delete.

Do not expose "query the vector table as a DBA" as the management
story. That breaks the abstraction the rest of this chapter spends on
purpose.

## Ingestion: from raw files to vectors

With indexes as the organizational unit, ingestion is the path that
turns a source file into embedded, searchable chunks. Stages are
strict: detect format, extract text and structure, capture metadata,
chunk per index config, embed through the Model Service, write the
vector store. Output of one stage is input to the next. There is no
"smart blob" that does all five in a vendor-shaped function you cannot
test.

### The challenge of diverse formats

Organizations do not file knowledge as a single MIME type. Policies are
PDFs. Product docs are Markdown. Help Center articles are HTML. Internal
procedures are Word. The pipeline has to emit **clean text plus
structure** regardless.

This is harder than a file extension check:

- **PDF** is a print format. Two columns interleave. Tables flatten.
  Headers and footers repeat. You will extract garbage unless the parser
  is allowed to be format-specific.
- **Word** carries comments, tracked changes, and styles next to the
  words you wanted.
- **HTML** wraps the article in nav, ads, and cookie banners.

Without a shared pipeline, Support writes a PDF extractor and Legal
writes a worse one. Neither team wanted that job. The platform's bet is
**one well-tested parser set**. Workflow authors call ingest and do not
learn `pypdf` edge cases.

If a format cannot be parsed into sections you trust, fail the job with
an error a human can act on. Silent "we ingested 12 pages of nav chrome"
is a retrieval bug that looks like a model bug.

### Pipeline architecture

A useful picture of the five stages:

```
  source bytes
       |
       v
  (1) FORMAT DETECTION     pdf | docx | html | md | txt
       |
       v
  (2) TEXT EXTRACTION      sections: heading, level, page, content
       |
       v
  (3) METADATA             auto (name, pages, words, time)
                           + caller (dept, type, author, version, tags)
       |
       v
  (4) CHUNKING             strategy from IndexConfig
       |
       v
  (5) EMBEDDING            Model Service, model frozen on the index
       |
       v
  vector store insert      text + vector + metadata + ids
```

Structure from (2) is not vanity. Structure-aware chunking later *needs*
headings and section breaks. If you flatten to a single string in stage
2, you threw away the only free signal the document had.

Keep the pipeline **synchronous in logic** even when you run it
asynchronously as a job. Same stages, same failure points, same
idempotency story (replace-on-reingest). Async is scheduling, not a
second algorithm.

### Format detection and text extraction

Detection and extraction are a pair. Identify the type, then hand bytes
to a parser that knows that type. The parser's job is an extracted
document: a list of **sections** (content, optional heading, heading
level, optional page) plus parser-owned metadata.

Why sections instead of one string? Because "Return Policy for
Electronics" is a unit a human already named. Chunkers that respect that
unit retrieve better than chunkers that cut at token 512 in the middle
of a fee schedule.

Detection should not trust the filename alone. `policy.pdf.exe` is not a
joke in every org. Bytes and declared type both matter; when they
disagree, fail closed.

Parsers are adapters in the same sense as model providers in
[chapter 3](../3-model-service/): swap the PDF library without changing
`ingest`. The Data Service owns the **interface** (`parse(bytes) ->
sections`), not a religion about which extractor is best this year.

### Metadata: the filtering foundation

Every document has attributes that are not its prose. Some are automatic
at parse time: filename, page count, word count, ingestion timestamp.
Some are supplied by the caller: department, document type, author,
version, tags, audience.

Vector search alone cannot tell a customer-facing reset guide from an
internal engineering runbook when both describe the same buttons.
**Metadata filters** make that distinction. A workflow that always
passes `audience=customer` never sees the internal chunk.

Metadata is not only `WHERE`. It also:

- **Attributes the answer** — "According to the Customer Service Policy,
  updated January 2025…"
- **Feeds freshness** — recency as a rank signal or a hard filter
  (`ingested_at > …`).
- **Makes audit possible** — list what is in the index without reading
  every chunk.

Carry metadata **onto every chunk**. A hit that cannot name its source
document is a citation you cannot defend. Filters that exist only on the
parent row and not on the chunk row will be forgotten under load.

Do not overload metadata with the full text. It is a sidecar, not a
second corpus.

### Chunking: breaking text into retrievable pieces

This is where retrieval quality is won or lost. Too large: the fact you
need is diluted. Too small: the fact has no context. Split a sentence in
half: you retrieve a riddle.

Parameters live on the **index** so the choice is consistent. The
platform should offer more than one strategy because corpora disagree.

A chunk is not a string. It is text plus enough pointer to reconstruct
provenance: heading, start/end offsets, document id, inherited metadata.

**Fixed-size (with overlap).** Cut on a token (or character) budget.
Without overlap, a sentence about restocking fees that straddles the
cut exists in no chunk as a complete thought. With overlap (the book's
order of 50 tokens is a starting point, not a law), the boundary
sentence appears intact in at least one chunk. You pay duplicate
storage. You buy recall at edges.

```
  tokens: |---- 512 ----|---- 512 ----|---- 512 ----|
  no overlap:        ^cut splits "defective items may be returned"
  overlap ~50:          |--shared--|
                        complete sentence lives in chunk 2
```

Fixed-size is content-blind. It does not care about paragraphs. Good
enough when topics happen to align. Wrong when precision is the product
(legal, clinical, safety).

**Recursive splitting.** Prefer large semantic separators, then fall
back. Typical ladder: paragraphs (`\n\n`), then lines (`\n`), then
sentences, then tokens. Pieces that already fit become chunks. Pieces
that do not recurse. This is a strong **default** because messy PDFs and
plain-text exports often have no trustworthy heading tree. You do not
need parser structure. You need *some* hierarchy of breaks.

**Structure-aware.** If extraction gave you "Return Policy for
Electronics" as a section, keep that section as one chunk when it fits
the token limit. If it does not, fall back to fixed or recursive *inside
the section*, not across the next heading. This is the strategy that
makes the parse work in stage 2 worth doing.

Pick by corpus, not by blog post:

| Corpus | Bias |
|---|---|
| FAQs, short articles | Fixed or recursive, modest size |
| Manuals with real H1/H2 | Structure-aware |
| Dumps and legacy TXT | Recursive |
| Tables / API specs | Often smaller chunks; consider table-aware parsers first |

A chunk that splits the only sentence containing the answer cannot be
saved by a better embedding model or by RRF. Fix the splitter before you
buy another vector database.

### Embeddings: reuse the Model Service

Chunks become vectors here. Conceptually: similar meanings, nearby
points. Operationally: **one model per index**, every chunk and every
query.

The platform does not hard-code the vendor model. Teams pick what fits
the content. The platform owns versioning, provider abstraction, retries,
and **cost tracking**. That is why embeddings go through the
[Model Service](../3-model-service/), not a second `openai.embeddings`
client in the Data Service. Fallback and dollar attribution already
exist. Duplicating them is sprawl with extra floating point.

Dimensionality is a knob with a bill. Higher dimensions can capture
finer distinctions and cost storage, memory, and latency on every
search. Some models let you shrink dimensions. That choice belongs in
index config because it affects every row you will ever insert.

Batch embed on ingest. Embed the **query at search time** with the same
model. A mismatch is not a warning; it is invalid geometry.

Watch the invoice. Ingest of a 2,000-page PDF is an embedding job, not
a free side effect of "we have RAG now." Observability in
[chapter 7](../7-observability/) should see those tokens as Data Service
work, not "misc AI."

### Document lifecycle

Ingestion is not a wedding. Policies update. Docs revise. Articles get
rewritten. The simple production pattern is **replace-on-reingest**:
same document id ingested again means delete existing chunks and vectors
for that id, then run the pipeline on the new bytes.

You re-embed unchanged paragraphs. That wastes some money. Diff-based
updates look clever until section boundaries shift and you keep orphan
vectors or drop a moved heading. For most corpora, re-embedding one
document is cheaper than debugging a partial update.

Identity must be stable. If every upload generates a new id, you never
replace; you accumulate ghosts. Prefer an explicit `document_id` from
the caller (CMS id, git path, policy number) and fall back to a
deterministic hash of a stable name, not of bytes (bytes change every
revision).

Retraction is a first-class event: delete the document (next section),
do not "ingest empty" and hope.

### Document management

Once files flow in, teams ask ordinary questions: what is in this index?
Did the quarterly report finish? Legal retracted a PDF; remove it. How
many chunks did the 200-page manual produce?

If the only answers live in raw SQL against the chunk table, the
platform abstraction has already leaked. Expose:

- **List documents** in an index (metadata, not every vector).
- **Get document** — metadata plus chunk count, ingest time, status.
- **Delete document** — chunks, vectors, keyword tokens for that id.

These are how you operate a library. Search hits are how you *use* one.
Do not make operators grep embeddings to see if ingest succeeded.

### Asynchronous ingestion

Synchronous ingest is fine for a two-page FAQ. A 2,000-page PDF can take
minutes. A thousand-file backfill can take hours. Blocking the workflow
that kicked ingest is how you teach teams to bypass the platform.

Accept the bytes, return a **job id**, process in the background. Poll
(or later, a callback) for status. States that match how humans debug:

```
  queued -> processing -> completed
                       \-> failed  (error that names the stage)
```

Progress that updates per stage (parsed, chunked, embedded, stored) is
worth more than a spinner. On success, the job points at `document_id`.
On failure, do not leave a half-written document searchable. Failed jobs
should not leak partial chunks; replace-on-reingest and transactional
inserts are how you keep that invariant.

Async is the same pipeline as sync. If you fork a "fast path" that skips
metadata, you will debug two systems.

## Vector storage and search

Ingest ends with "put these vectors somewhere." The platform's job is a
**storage abstraction**: write during ingest, read at query time,
backends swappable. Same pattern as session storage in chapter 4:
abstract interface, one serious implementation (here, Postgres +
pgvector), room for others. The operations are not the same — sessions
are rows by id; vectors are similarity over arrays — but the *adapter*
idea is identical.

### Vector store interface

Two jobs: store, search. Keep them on one interface so ingest and query
cannot drift across backends.

**Writes**

- **Insert** — chunks + embeddings + metadata for a document into an
  index. Return how many rows landed.
- **Delete by document** — all chunks for that id (lifecycle).
- **Delete index** — everything in that collection.

Inserts should be **index-scoped**. A backend that is a single global
ANN index with a metadata filter pretending to be isolation is a
different, weaker contract. Prefer real partitioning (table, namespace,
or collection per index) so a bug cannot search across tenants.

**Reads**

- **Search** — given a query *embedding*, an index name, `top_k`,
  optional metadata filters, optional score threshold. Return a list of
  hits: chunk text, score, document id, metadata, maybe heading.

Same `SearchResult` type for vector and (later) keyword paths. Fusion
and the SDK should not care which path produced a row.

### Choosing a backend

The market is still settling. Two families:

- **Extensions on databases you already run.** pgvector on PostgreSQL is
  the teaching default in this book for a reason: you likely already
  store sessions there. Backups, IAM, and on-call stay one system. For
  many orgs, on the order of millions of vectors, this is enough if you
  index and memory-size honestly.
- **Purpose-built vector databases.** Query planners and storage laid
  out for ANN. Further scale, extra ops (or a fully managed bill).
  Pinecone-class services trade control for not running the thing.

Choose on **operations and isolation**, not on blog-bench QPS.

| You already… | Lean toward |
|---|---|
| Run Postgres well, < large-millions vectors | pgvector |
| Need multi-region ANN as a product | Dedicated engine |
| Cannot operate another stateful system | Managed vector **or** pgvector on the DB you already pay people to run |
| Must keep vectors next to relational metadata and transactions | pgvector (or similar) |

The interface exists so this choice is not a rewrite of ingest. Do not
let the SDK speak Pinecone in one workflow and `<=>` in another.

### A pgvector-shaped example

You do not need the book's CREATE TABLE memorized. You need the **row
shape**:

- chunk id, document id, index name
- chunk text
- embedding (`vector(N)` with **N frozen to the index**)
- metadata (JSONB)
- timestamps

Indexes you actually want: by `document_id` (deletes), by `index_name`
(scoped search), and an ANN index on the embedding column (the reason
you installed the extension). A single `vector(1536)` column for every
index only works if every index shares dimensions. If indexes can
disagree on model, your schema must not lie — separate tables, or a
schema per index, or a design that rejects mixed dimensions. Silent
truncate/pad is corruption.

Similarity at query time is a SQL operator over that column, filtered by
`index_name` and JSONB metadata, limited to `top_k`, optionally dropping
rows below a threshold. The Data Service, not the workflow, writes that
SQL.

HNSW vs IVF choices are backend tuning. Put them in ops runbooks, not in
the gRPC contract.

### Search orchestration

Workflows search with **text**, not with vectors. Orchestration is the
method that:

1. Loads index config (hence the embedding model).
2. Embeds the query through the same Model Service path as ingest.
3. Calls `vector_store.search` with filters and `top_k`.
4. Returns hits the workflow can stuff into context or cite.

```
  search(index, "laptop return window", top_k=5, filters={...})
       |
       v
  get IndexConfig.embedding_model
       |
       v
  Model Service.embed(query)     // not a second provider SDK
       |
       v
  VectorStore.search(index, embedding, top_k, filters, threshold)
```

If you embed the query with a different model than ingest, you will get
confident nonsense. If you skip index lookup and pass a global embedding
function, you have already broken isolation's config story.

`score_threshold` is how you stop injecting unrelated chunks because
`top_k=5` always returns five rows. Empty results are a valid answer:
the library does not contain this. Workflows should be allowed to say
"I don't know" instead of quoting a 0.11 similarity.

## Hybrid search: keywords on purpose

Semantic search matches "laptop return window" to "electronics refund
period." That is the point of embeddings. It is the wrong tool when the
user's query *is* an identifier: `ERR-4012`, a SKU, `HIPAA 164.512(a)`.
Embeddings smear those strings into a cloud of "errors" and "problems."
The hit you needed had the exact token.

Keyword search has the opposite failure: paraphrase misses, exact tokens
hit. Production corpora need **both**. Hybrid is not a fashion. It is
how mixed questions (natural language + codes) get one ranked list.

See the RAG rows in [`TRADEOFFS.md`](../../TRADEOFFS.md): vector-only
misses identifiers; keyword-only misses paraphrase; hybrid plus fusion
costs two retrievals and some tuning. Isolated indexes still apply — you
fuse within an index, not across the company.

### Keyword search on the platform

Extend the vector store interface with **optional** `keyword_search`.
Same index scope, same metadata filters, same `SearchResult` type.
Difference: the argument is a **raw query string**, not an embedding.

It should not be a required abstract method on every backend. A pure ANN
store might not have a text postings list. Default: raise a clear
"this backend has no keyword path." Hybrid orchestration then either
skips fusion (degraded, logged) or refuses to advertise `HybridSearch`
for that backend. Silent vector-only while the API name says hybrid is
a lie.

Keep the method on the store, not as a second microservice. Fusion needs
both lists in one place.

### Keyword search in Postgres

Dedicated engines often rank with **BM25** (term frequency, inverse
document frequency, length normalization). PostgreSQL's built-in
full-text search is not BM25. It is still a pragmatic start: `tsvector`
of chunk text (stemming, stop words), GIN index, `@@` match, `ts_rank`
for an in-list order.

A generated `tsvector` column maintained from `chunk_text` means ingest
does not grow a second writer. You already stored the text. The database
derives tokens.

Limitations to say out loud: ranking is weaker than BM25; language
config (`english` vs others) matters; identifiers with punctuation
(`ERR-4012`, section numbers) may need `simple` config or `phraseto`
care so you do not stem the code into mush. If keyword quality is the
product, evaluate extensions that implement real BM25. If hybrid is a
safety net under vectors, built-in FTS is an honest v1.

Do not tokenize only at query time with `LIKE '%ERR-4012%'`. That is a
full scan wearing a costume.

### Reciprocal Rank Fusion

You now have two lists. Vector scores are roughly cosine in a bounded
range. Keyword scores are a different universe (unbounded ranks, or
`ts_rank` in its own scale). **Do not add the scores.** Normalizing them
into one numeric religion is brittle across corpora.

**Reciprocal Rank Fusion** ignores magnitudes and uses **ranks**. Each
appearance of a chunk contributes `1 / (k + rank)` with `k` a constant
that keeps the #1 hit from dominating. The original RRF work's `k=60`
is the default people copy because it is boring and works widely. A
chunk near the top of *both* lists rises. A chunk that appears once,
low, stays low.

```
  vector ranks:  [A, B, C, D]
  keyword ranks: [C, A, E, F]
  RRF(C) = 1/(k+1) + 1/(k+3)   // strong on both
  RRF(B) = 1/(k+2)             // vector only
```

No training. No score calibration. You still must choose `top_k` per
leg (often fetch more than you return, then fuse, then cut). You still
must not fuse across indexes.

RRF is not magic if both legs are wrong. Garbage parsers in, fused
garbage out.

### Putting hybrid together

One entry point, two legs, one list out:

```
                 hybrid_search(index, query, top_k, filters)
                                  |
                 +----------------+----------------+
                 v                                 v
        embed(query, index.model)          tsquery(query)
                 |                                 |
                 v                                 v
        vector search (cosine)             keyword search (FTS)
                 |                                 |
                 +----------------+----------------+
                                  v
                               RRF merge
                                  v
                          SearchResponse (one ranking)
```

Run the legs in parallel when you can; ingest already paid for both
indexes on the same rows. The caller never sees two lists unless you
are debugging. Debugging should log both rankings and the fused order —
otherwise you cannot tell whether a miss was vector, keyword, or fusion.

Workflows that know the query is *only* an error code can still call
keyword search. Workflows that know it is *only* paraphrase can call
vector search. Hybrid is the default for mixed traffic, not a
requirement to always spend twice.

## Service contract and complete retrieval flow

Like Model and Session, Data is a service with a **contract** (gRPC in
the book) and an SDK client. Three groups of RPCs:

**Index management** — create, get, list, delete. Config and isolation
live here.

**Document ingestion** — ingest (returns a job in the async design),
get ingest job, delete document. List/get document metadata belongs
with this group even if the protobuf splits hairs.

**Search** — `Search` (vector) and `HybridSearch` (vector + keyword +
RRF). Same response message: ranked chunks plus scores and source
metadata.

The contract is how you stop each language binding from inventing a
fourth notion of index. SDK: `platform.data.ingest(...)` and
`platform.data.hybrid_search(...)` should not mention pgvector.

End-to-end, Maria's "return policy for electronics?" from
[chapter 1](../1-why-a-platform/) looks like this — still not
`openai.chat()`:

1. Gateway authenticates; workflow starts.
2. Session loads that she already talked about a laptop.
3. **Data:** hybrid search on `support.policies` with
   `audience=customer`. Chunks cite the current PDF, not last year's
   FAQ pasted in a prompt.
4. Model generates with those chunks in context.
5. Observability records retrieval latency, embedding tokens, hit ids
   — so a bad answer can be blamed on a miss, a bad chunk, or the
   model, not "AI."

If step 3 searches the wrong index, guardrails in
[chapter 6](../6-tools-and-guardrails/) will not save you. Isolation is
a data-plane control, not a content filter.

### What this service is not

It is not conversational memory. "Our enterprise discount" said in chat
belongs in Session (and maybe model-managed memories), not in a PDF
index you forgot to update.

It is not an agent loop. Retrieving twice because the first hits were
weak is **agentic RAG** — Agents track, chapter 6. This service offers
search as a reliable call. The workflow or agent decides whether to
call it again.

It is not evaluation. Chunk quality and "did the answer use the hit"
are [chapter 7](../7-observability/) and the Agents eval chapter. You
still instrument ingest and search so those judges have something to
score.

## Check yourself

1. A user says "as I mentioned yesterday, our enterprise discount."
   Which service should answer, and what goes wrong if that fact lives
   only as a chunk in a vector index of PDFs?
2. Support and Legal share a platform. Give one *safety* failure of a
   single shared index, and one *quality* failure of forcing both teams
   onto the same embedding model and chunk size.
3. Why is the embedding model immutable on an index? What is the honest
   migration path when a team wants a new model?
4. Sketch the five ingest stages for a two-column PDF. Where does a
   header/footer bug become a retrieval bug that looks like
   hallucination?
5. Fixed-size vs recursive vs structure-aware: pick one corpus you
   actually have (or Sam's returns PDF) and argue which splitter you
   would freeze on the index — including what you would *lose* with the
   other two.
6. Why do embeddings and query vectors go through the Model Service
   instead of a private OpenAI client inside Data? Name two platform
   behaviors you would otherwise duplicate.
7. Replace-on-reingest vs diff-update: which failure mode of diffs is
   this chapter willing to pay extra embedding cost to avoid?
8. A search for `ERR-4012` returns semantically adjacent "error
   handling" essays. Which leg is missing, and why is adding cosine
   scores to `ts_rank` a bad merge?
9. Write RRF for a chunk that is rank 1 on keywords and rank 8 on
   vectors (`k=60`). Why can a mid-ranked pair outrank a single #1?
10. Draw Maria's question through gateway, session, data, and model.
    Where do metadata filters belong, and what happens if the workflow
    is allowed to pass any `index_name` string?

Continue to [Tools and guardrails](../6-tools-and-guardrails/).
