# 5. The Data Service

Companion notes for **Chapter 5** of *Designing AI Systems* (Suhas Suresha and Dewang
Sultania; Manning MEAP).

This chapter establishes the Data Service: isolated indexes, one ingestion pipeline, and
hybrid search the organization can operate. Afterward you can name an index as a product
boundary, walk a file through detect, extract, metadata, chunk, and embed, and say why a
keyword leg and Reciprocal Rank Fusion sit next to vectors. Sessions keep what Maria said
yesterday. This service keeps what Legal published last Tuesday. Paste a PDF into the
prompt, or let every squad own a chunker, and you pay tokens, noise, stale pages, and four
incompatible notions of a document id.

## 1. Passages, not manuals

Somewhere in a 40-page returns PDF, page 3 says electronics can come back within 30 days
with a receipt and the original packaging, plus a restocking fee if the item was opened. The
customer asks, "What happens if I return a laptop I already opened?"

* **The mismatch**: Keyword search looks for *laptop*, *opened*, and *return*. The paragraph
  may say *electronics* and *original packaging*. Exact tokens miss the paraphrase.
  **Semantic search** compares meanings, usually as dense vectors, at the grain of a
  passage.
* **Why not the whole file**: A ten-word question against a 50-page blob mixes return
  windows, warranty text, and shipping tables. A passage has to be small enough to be one
  idea and large enough to still be that idea. If you cannot name the passage that should
  answer a real question, you are not ready to tune `top_k`. The library is not searchable
  yet.
* **Who does the work**: A workflow calls `ingest` and `search`, or `hybrid_search`.
  Parsing, overlap, and the vector backend are platform work. Which index, and which
  filters, are product work.

Three shortcuts fail for boring reasons.

1. **Paste the PDF into the prompt**: You hit the window, pay for every token, and still
   miss the paragraph in the noise. An update means a deploy or a prompt edit.
2. **One regex over raw bytes**: Columns, headers, and "laptop" versus "electronics" break
   it.
3. **A notebook chunker per squad**: Four embedding models, four document ids, four ways to
   forget a retracted policy.

* **An index is the boundary**: Embedding model, chunker, and corpus stay frozen together. A
  support query must not retrieve a sealed legal memo. Vectors from two models must not
  share a similarity operator. "Our enterprise discount," said in chat, does not belong in a
  PDF corpus that never heard Maria. That fact lives in the session store.

## 2. Indexes

An **index** is a named, isolated collection with its own embedding configuration and search
behavior. A transcript belongs to a session. A corpus belongs to an index. Product docs, HR
policy, and runbooks are three names. Isolation is the feature.

```
  search(index=support_kb)              search(index=legal_kb)
  embed: small                          embed: legal-ft
  chunk: recursive                      chunk: structure
  docs: runbooks                        docs: statutes
```

* **Scoped search**: Stronger than a `WHERE team =` filter someone must remember. Filters
  fail open when omitted. The candidate set itself is per index. "How to reset a device"
  does not rank a data-retention clause, because those rows are not candidates. A single
  company brain with `chunk_size=512` forces a FAQ and a statute through the same grain.
* **Create-time contract**: A stable **name** (`support.troubleshooting`, not `kb2`). An
  **embedding model and its dimensions**. A **chunking strategy and sizes**. An optional
  **metadata schema** for the filter keys you expect, such as `audience=customer|internal`.
* **Immutable vectors**: Different models live in different spaces. Mixing them corrupts
  nearest-neighbor search, and the scores still look like numbers. Reject "switch this index
  to `text-embedding-3-large`." Create a new index, re-ingest, cut search over, delete the
  old one. That annoyance is cheaper than undiagnosable 0.72 similarities. Overlap has the
  same rule: change it only with a re-ingest, or half the corpus is the old grain.
* **Metadata as a mild type system**: If every runbook must carry `audience`, ingest fails
  when the caller omits it.
* **Lifecycle**: **Create** allocates the collection and freezes config. **Get** returns
  config and stats: document count, chunk count, `created_at`, `last_ingested_at`. **List**
  returns what this caller may see, not every tenant. **Delete** drops documents, chunks,
  vectors, and keyword postings. Treat it as destruction. `last_ingested_at` older than 90
  days is stale knowledge as a timestamp, before it is a customer complaint. **Owner** is
  who is paged when embeddings fail, and who may delete.
* **Which names are legal**: A workflow that may pass any `index_name` the prompt
  interpolates is a confused deputy: the service uses a power the caller should not have
  had. Management is these RPCs. Querying the vector table as a database admin breaks the
  abstraction.

### Watch for

* One shared index plus a filter the caller forgot. Legal text shows up in a support answer.
* An index with no owner. Nobody is allowed to delete it, and nobody is paged when ingest
  fails.

## 3. Ingestion

Ingestion turns bytes into embedded chunks. Five stages. The output of one is the input of
the next. A single vendor call that hides all five cannot be tested.

```
  source bytes
       |
       v
  (1) FORMAT DETECTION    pdf | docx | html | md | txt
       |
       v
  (2) TEXT EXTRACTION     sections: heading, level, page, content
       |
       v
  (3) METADATA            automatic fields + caller fields
       |
       v
  (4) CHUNKING            strategy from the index config
       |
       v
  (5) EMBEDDING           Model Service, model frozen on the index
       |
       v
  insert                  text + vector + metadata + ids
```

* **One parser set**: Policies arrive as PDF, product docs as Markdown, help articles as
  HTML, procedures as Word. The pipeline emits clean text plus structure. PDF is a print
  format: two columns interleave, tables flatten, headers and footers repeat. Word carries
  comments, tracked changes, and styles beside the words you wanted. HTML wraps the article
  in navigation, ads, and cookie banners. Support should not grow a PDF extractor while
  Legal grows a worse one. A format you cannot turn into sections you trust fails the job
  with an error a human can act on.
* **Detect, then parse**: Do not trust the filename. `policy.pdf.exe` is a real file in some
  orgs. When bytes and declared type disagree, fail closed. The Data Service owns
  `parse(bytes) -> sections`. Swap the PDF library without changing `ingest`.
* **Sections are the unit**: Content, optional heading, heading level, optional page.
  "Return Policy for Electronics" is a unit a human already named. Flatten stage 2 to one
  string and structure-aware chunking has nothing left to respect.
* **Same stages when the job is async**: Asynchronous ingest is scheduling. Failure points
  and replace-on-reingest stay the same. A fast path that skips metadata is a second system.
* **Metadata on every chunk**: Automatic at parse time: filename, page count, word count,
  ingestion timestamp. From the caller: department, document type, author, version, tags,
  audience. Two chunks can describe the same buttons and still be a customer reset guide
  versus an internal runbook. `audience=customer` on the search is what keeps them apart.
  The same fields attribute the answer ("According to the Customer Service Policy, updated
  January 2025"), feed freshness (`ingested_at` as a rank signal or a hard filter), and let
  you list the library without reading every vector. A hit that cannot name its source
  document is a citation you cannot defend. Filters that live only on the parent row get
  dropped under load. The full text already sits on the chunk. Metadata stays a sidecar.

### Watch for

* Twelve pages of navigation chrome ingested as policy. Retrieval looks confident. The bug
  looks like the model inventing text.
* A repeated header or footer stored as its own chunk. The model cites the chrome.

## 4. Chunking

Retrieval quality is won or lost here. Too large, and the fact is diluted. Too small, and
the fact has no context. A sentence cut in half is a riddle. The parameters live on the
index so every document in that corpus shares them. A chunk is text plus provenance:
heading, start and end offsets, document id, inherited metadata.

```
  tokens: |---- 512 ----|---- 512 ----|---- 512 ----|
  no overlap:    ^cut splits "defective items may be returned"
  overlap ~50:      |--shared--|
                    the sentence survives in the next chunk
```

1. **Fixed-size, with overlap**: Cut on a token or character budget. Overlap on the order of
   50 tokens is a starting point, not a law. The boundary sentence then exists intact in at
   least one chunk. You pay duplicate storage. You buy recall at the cut. Use it when topics
   happen to line up with the window, including short FAQs. It breaks on legal, clinical,
   and safety text, because the cutter cannot see a fee schedule.
2. **Recursive splitting**: Try large separators, then smaller ones. A typical ladder is
   paragraphs (`\n\n`), then lines (`\n`), then sentences, then tokens. A piece that fits
   becomes a chunk. A piece that does not fit is split again. Use it as the default for
   messy PDFs and legacy text that have no trustworthy heading tree. It breaks when a real
   heading outline exists and you still slice a statute across sections.
3. **Structure-aware**: Keep "Return Policy for Electronics" as one chunk when it fits the
   token limit. If it does not, fall back to fixed-size or recursive *inside* that section,
   and stop at the next heading. Use it for manuals with real H1 and H2 headings. That is
   why stage 2 kept structure. It breaks when the parser's headings are garbage, such as a
   running header repeated on every page. You then freeze a bad outline.

| Corpus | Where to start |
|---|---|
| FAQs, short articles | Fixed-size or recursive, modest size |
| Manuals with real headings | Structure-aware |
| Dumps and legacy text | Recursive |
| Tables and API specs | Smaller chunks, after a table-aware parse |

* **The returns PDF**: Page 3 is one idea: 30 days, a receipt, original packaging, a
  restocking fee if the item was opened. Structure-aware keeps "Return Policy for
  Electronics" together when the section fits. Fixed-size can cut the fee sentence on a
  512-token boundary. Recursive keeps the paragraph when a blank line is real, and still
  slices across headings when the outline is junk. Freeze structure-aware only after you
  trust those headings.
* **The splitter outranks the model**: A chunk that cuts the only sentence with the answer
  cannot be rescued by a better embedding or by fusion. Fix the split before you buy another
  vector database.

## 5. Embeddings and document identity

Similar meanings land near each other. The operational rule is stricter: **one model per
index**, on every chunk and on every query.

* **Embed through the Model Service**: The same platform service that generates chat replies
  also embeds. Teams choose the model that fits the corpus. The platform already owns
  versioning, provider fallback, retries, and cost. A private `openai.embeddings` client
  inside Data duplicates that path and loses the dollar attribution. Ingest of a 2,000-page
  PDF is an embedding job. Those tokens are Data Service ingest, not miscellaneous model
  spend.
* **Dimensions are config**: Higher dimensions can separate finer meanings. They cost
  storage, memory, and latency on every search. Some models let you shrink the vector.
  Either choice is frozen on the index, because it is the shape of every row you will
  insert.
* **The query uses the ingest model**: Batch the chunks at ingest. Embed the question at
  search time on the same model. A mismatch is invalid geometry. The scores will still look
  plausible.
* **Replace-on-reingest**: The same `document_id` means delete that id's chunks and vectors,
  then run the pipeline on the new bytes. Unchanged paragraphs are embedded again. A diff
  looks cheaper until a section boundary moves, an orphan vector remains, or a relocated
  heading disappears. Re-embedding one document is usually cheaper than debugging a partial
  update.
* **Stable ids**: A fresh id on every upload accumulates ghosts. Prefer a caller-supplied
  `document_id`: CMS id, git path, or policy number. If you must derive one, hash a stable
  name. Do not hash the bytes. Bytes change on every revision. Retraction is delete. An
  empty ingest is a poor substitute.
* **Operate the library**: **List documents** returns metadata, not every vector. **Get
  document** returns metadata, chunk count, ingest time, and status. **Delete document**
  removes chunks, vectors, and keyword tokens for that id. If the only way to answer "did
  the quarterly report finish?" is SQL against the chunk table, the abstraction has leaked.
* **Jobs for long files**: A two-page FAQ can finish in the request. A 2,000-page PDF takes
  minutes. A thousand-file backfill takes hours. Accept the bytes, return a job id, and move
  through `queued`, then `processing`, then `completed` or `failed`. The failure names the
  stage. Progress that says parsed, chunked, embedded, or stored beats a spinner. Success
  points at `document_id`. Failure leaves nothing searchable. Transactional insert, and
  replace-on-reingest, are how a half-written document stays out of search.

### Watch for

* Padding or truncating vectors so two models share one column. That is corruption with a
  successful insert.
* A failed job whose early chunks are already searchable. The library cites a document that
  never finished.

## 6. Vector storage and search

Ingest writes vectors. Search reads them. One interface so the two paths cannot drift.
Sessions are rows fetched by id. Vectors are similarity over arrays. The adapter pattern is
the same idea: the SDK never speaks a backend dialect.

* **Writes are index-scoped**: **Insert** stores chunks, embeddings, and metadata for one
  document and returns how many rows landed. **Delete by document** removes that id.
  **Delete index** removes the collection. A global approximate-nearest-neighbor index with
  a metadata filter pretending to be isolation is a weaker contract. Prefer a real
  partition: a table, a namespace, or a collection per index.
* **Search**: A query embedding, an index name, `top_k`, optional metadata filters, an
  optional score threshold. Each hit carries chunk text, score, document id, metadata, and
  maybe a heading. Vector and keyword paths share one `SearchResult` so fusion sees one row
  shape.

1. **A database extension** (pgvector on PostgreSQL): The teaching default. Sessions may
   already live here, so backups, identity, and on-call stay one system. On the order of
   millions of vectors it is enough when the index and the memory budget are honest. It
   breaks when multi-region approximate nearest neighbor is the product you sell, and
   Postgres is not that product.
2. **A purpose-built vector database**: Storage and planners laid out for that search. You
   gain scale, and you take on another system or a managed bill. Pinecone-class services
   trade control for not running the engine. It breaks when vectors must sit in the same
   transactions as relational metadata and you already run Postgres well.

| You already… | Lean toward |
|---|---|
| Run Postgres well, under the large millions of vectors | pgvector |
| Need multi-region approximate nearest neighbor as a product | A dedicated engine |
| Cannot operate another stateful system | Managed vector service, or pgvector on the database you already staff |
| Need vectors beside relational metadata and transactions | pgvector, or something with the same shape |

* **The row**: Chunk id, document id, index name, chunk text, `vector(N)` with N frozen to
  that index, metadata as JSON, timestamps. You want a lookup by `document_id` for deletes,
  a lookup by `index_name` for scoped search, and an approximate-nearest-neighbor index on
  the embedding. One shared `vector(1536)` column works only when every index has 1536
  dimensions. Mixed models need separate tables, a schema per index, or a hard reject. The
  Data Service writes the SQL: a similarity operator on that column, filtered by index name
  and metadata, limited to `top_k`, optionally dropping scores under the threshold. HNSW
  versus IVF is an operations choice. It does not belong on the service contract. Workflow
  code does not speak Pinecone in one app and `<=>` in another.

Workflows pass text. The service loads the index, embeds with that index's model, and
searches.

```
  search(index, "laptop return window", top_k=5, filters={...})
       |
       v
  index embedding model -> Model Service.embed(query)
       |
       v
  VectorStore.search(index, embedding, top_k, filters, threshold)
```

* **A threshold beats a forced five**: `top_k=5` always returns five rows. `score_threshold`
  drops the unrelated ones. An empty list is a valid answer: the library does not contain
  this. Quoting a 0.11 similarity is how "I don't know" gets replaced by a confident miss.
  Skip the index lookup, embed with a global function, and you have already broken the
  per-index model.

## 7. Keyword search and reciprocal rank fusion

Vectors match "laptop return window" to "electronics refund period." An identifier is the
wrong job for that match. `ERR-4012`, a SKU, and `HIPAA 164.512(a)` smear into "errors" and
"problems." Keyword search hits the exact token and misses the paraphrase. Mixed questions
need both legs, fused inside one index.

* **keyword_search**: Optional on the same store interface. Same index, same metadata
  filters, same `SearchResult`. The argument is the raw query string. A pure vector backend
  may have no postings list, so the method is not required on every implementation. The
  honest default is a clear error: this backend has no keyword path. Hybrid then skips
  fusion and logs the degrade, or it refuses to advertise `HybridSearch`. A response named
  hybrid that ran only vectors is a lie. Both lists have to meet in this process. A second
  microservice just to hold keywords splits the fusion.
* **BM25**: The ranker many lexical engines use. It combines term frequency, inverse
  document frequency, and length normalization. Use it when keyword quality is the product.
  Built-in Postgres full text is a weaker rank than BM25, so an extension that implements
  BM25 is the upgrade, not the first version.
* **Postgres full text as v1**: A generated `tsvector` on `chunk_text` (stemming and stop
  words), a GIN index, `@@` for the match, `ts_rank` for order inside the list. Ingest does
  not grow a second writer. The database derives tokens from text you already stored. Honest
  when the keyword leg is a safety net under vectors. The language config matters (`english`
  versus another).
* **Identifiers**: `ERR-4012` and section numbers can be stemmed into mush. A `simple`
  config, or `phraseto_tsquery`, keeps the token intact. `LIKE '%ERR-4012%'` is a full scan
  wearing a costume.
* **Ranks, not added scores**: Cosine lives in a bounded range. `ts_rank` lives on its own
  scale. Adding them mixes units. Rescaling both into one number is brittle from corpus to
  corpus.

**Reciprocal Rank Fusion** ignores magnitudes. A chunk at rank `r` contributes
`1 / (k + r)`. The constant `k` stops rank 1 from dominating. The usual default, from the
original RRF work, is `k=60`.

```
  vector ranks:  [A, B, C, D]
  keyword ranks: [C, A, E, F]
  RRF(C) = 1/(k+1) + 1/(k+3)     near the top of both
  RRF(B) = 1/(k+2)               vector only
```

A chunk at keyword rank 1 and vector rank 8 scores `1/(60+1) + 1/(60+8)`, that is
`1/61 + 1/68`. A lone rank 1 on one leg scores only `1/61`. Rank 4 on both legs scores
`1/64 + 1/64`, and that sum beats the lonely first place. No training and no score
calibration. Fetch more than you return on each leg, fuse, then cut. Fusion stays inside
one index. Two wrong lists fuse into a wrong list. A bad parser is still a bad parser
after RRF.

* **hybrid_search**: One entry point. It embeds with `index.model` and builds the text query
  for the same index. Vector search (cosine) and keyword search run as two legs. They can
  run in parallel, because ingest already built both structures on the same rows. Fusion,
  then cut, produces one `SearchResponse`.
* **Which leg**: An error code can call keyword search alone. A paraphrase can call vector
  search alone. Hybrid is the default for mixed traffic. You pay for two retrievals when the
  query shape needs both.
* **Debug the fuse**: Logs keep both rankings and the fused order. Otherwise a miss cannot
  be blamed on the vector leg, the keyword leg, or RRF.

## 8. The contract in one retrieval

The service contract is what stops every language binding from inventing a fourth kind of
index. `platform.data.ingest(...)` and `platform.data.hybrid_search(...)` do not leak
pgvector.

* **Index management**: Create, get, list, delete. Configuration and isolation live here.
* **Documents**: Ingest returns a job id in the async design. Get ingest job, list
  documents, get document, delete document.
* **Search**: `Search` is the vector leg. `HybridSearch` is vector, keyword, and RRF. Both
  return the same message: ranked chunks, scores, and source metadata.

Maria asks, "return policy for electronics?"

1. The gateway authenticates. The workflow starts.
2. The session store still knows she already talked about a laptop.
3. Data runs hybrid search on `support.policies` with `audience=customer`. The chunks cite
   the current PDF. Last year's FAQ stays out of the prompt.
4. The model generates with those chunks in context.
5. The trace records retrieval latency, embedding tokens, and hit ids. A bad answer can be a
   miss, a bad chunk, or the model.

* **The filter is on the search**: `audience=customer` is enforced in the candidate set.
  Searching the wrong index is not something a later content filter repairs. Calling search
  again because the first hits were weak is the workflow's decision. This service returns a
  reliable ranking. Whether the answer used the hit is a later judgment. You still record
  ingest and search so that judgment has ids to score.

## Check yourself

1. A user says "as I mentioned yesterday, our enterprise discount." Which store should
   answer, and what goes wrong if that sentence exists only as a chunk in a PDF index?
2. Support and Legal share a platform. Name one safety failure of a single shared index, and
   one quality failure of forcing both onto the same embedding model and chunk size.
3. Why is the embedding model immutable on an index? What is the migration when a team wants
   `text-embedding-3-large`?
4. Walk a two-column PDF through the five ingest stages. Where does a repeated header become
   a retrieval bug that looks like a hallucinated answer?
5. For the returns PDF, which splitter would you freeze on the index: fixed-size, recursive,
   or structure-aware? What do you lose with each of the other two?
6. Why do chunk vectors and query vectors go through the Model Service instead of a private
   embeddings client inside Data? Name two platform behaviors you would otherwise rebuild.
7. Replace-on-reingest spends embedding money on unchanged paragraphs. Which diff-update
   failure is that cost paying to avoid?
8. A search for `ERR-4012` returns essays about error handling. Which leg is missing, and
   why is adding cosine to `ts_rank` a bad merge?
9. A chunk is rank 1 on keywords and rank 8 on vectors, with `k=60`. Write its RRF score.
   Why can a pair of mid ranks beat a single rank 1?
10. Walk Maria's return-policy question through gateway, session, data, and model. Where do
    metadata filters belong, and what happens if the workflow may pass any `index_name`?
