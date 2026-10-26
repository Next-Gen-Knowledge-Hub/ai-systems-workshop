# 6. Memory and knowledge RAG

Companion notes for **Chapter 6** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is how an unbounded store reaches one model call. The call has a context window. Manuals, tickets, and the record of what happened with a user do not. After this chapter you can name what you retrieved (a chunk, a keyword hit, a vector neighbor, a fused list, a graph neighborhood, or a memory row), how the score was computed, and what the agent should say when the hit list is empty.

## 1. The library, the lived record, and the window

External stores can grow without a fixed limit. One model call cannot. **Retrieval** searches the large store and seats a small slice in this prompt.

```
UNBOUNDED                         THIS CALL
wiki, PDFs, tickets               system and persona
vector index                      tool schemas
keyword / BM25 index   retrieve   retrieved slices
graph or SQL           ------->   this session's chat
prior sessions                    working notes
```

* **Knowledge**: The library. Manuals, policies, catalogs. It changes slowly. You ingest it as chunks plus metadata. The question it answers is what the corpus says.
* **Memory**: The lived record. This session's turns, facts from last session, preferences, episodes. The question it answers is what happened with this user.
* **Same mechanics, different hygiene**: Embed, search, and fetch show up in both. Freshness, who is allowed to write, and how you delete do not. One collection for chat logs and policy PDFs treats a preference as a regulation, and a regulation as a preference.

| | Knowledge | Memory |
|---|---|---|
| What it holds | Manuals, policies, catalogs | Turns, preferences, episodes |
| How fast it changes | Slowly | As the conversation happens |
| The question | What does the corpus say? | What happened with us? |

* **A product, not a flag**: "We added RAG" means an ingestion path, a query path, and a rule for thin or empty hits. "The agent remembers" means a write, a read, and a way to forget, usually across more than one store. A bigger prompt is not that write path.

### Watch for

* A large window is still not an index. A million tokens of handbook can fit and still fail: attention is uneven, every token is paid for, and a contradictory paragraph still gets a vote.
* Retrieval chooses. It does not hoard. The window behaves like RAM and the index like disk. Loading the corpus on every turn spends the budget you needed for the question.

## 2. What can live outside the weights

Asked for a private or recent fact, the model invents one when that fact exists only as a guess in the weights. Keep the fact outside. At call time, fetch a small relevant set into the prompt, or into a tool result the prompt can see.

* **Retrieval**: The step that turns a large external state into tokens this call can use. It is a mechanism you build, not a switch on the model.
* **Unstructured text**: Tickets, transcripts, notes, preferences written as sentences. The usual path is embed and search.
* **Structured state**: Rows and files you would query with SQL, a filesystem, or an API. The usual path is a tool. Production questions use both.
* **A context policy is a different job**: Stuffing the last 200 turns into the window works until the transcript or the handbook no longer fits. Truncation and summarization of the live chat are a token budget. The agent question is which tool to call, with which query, and what to do when the hit list is empty.

Name the bucket before you pick a store, a retention rule, or a tool.

1. **Documents**: Handbooks, specs, contracts. Knowledge.
2. **Operational records**: Error catalogs, SKUs, runbooks. Knowledge whose useful tokens are exact: codes and ids. A paraphrase index will smear them.
3. **Conversation**: This thread is short-term and usually already in the window. Earlier threads are long-term and have to be retrieved. Memory.
4. **Experiential traces**: "Last time we refunded this waybill we needed a manager code." Memory that behaves like a procedure the agent already learned.
5. **User facts**: Diet, timezone, account tier. Memory you should not ask for again next session.

## 3. RAG is two phases

**RAG** (retrieval-augmented generation) is an ingestion pipeline and a query pipeline. A demo of the second phase is half the machine.

```
INGESTION                         RETRIEVAL, PER QUESTION
load documents                    embed the query
split into chunks                 search nearest chunks
attach metadata                   optional keyword pass
embed each chunk                  put hits in the prompt
upsert into a store               generate, then check grounding
```

* **Embedding model**: Maps text to a dense vector so that similar meanings sit near each other. This model is not the one that writes the answer.
* **Generator**: The chat or agent model. Swapping the embedder without re-indexing searches a space the documents do not occupy. The two upgrades are different jobs.
* **Name the missing half**: A single PDF, and a question copied from chunk 4, only shows that a verbatim span can be found. Draw both phases before you call the system done.
* **What you should be able to point at**: The chunker. The embedder. The store. The `k`. The sentence the agent says when there are no hits. The grounding rule for hits that exist and still do not contain the claim. Without that last rule, retrieval is a suggestion the generator may ignore.

Chunk size is a bet on the smallest unit that still answers a question.

* **Too small**: A table cell without its header. A "yes" whose policy was in the previous chunk.
* **Too large**: Five topics in one vector. Search hits the blob, and the generator drowns in neighbors.
* **No overlap**: The sentence that crossed the cut is in neither neighbor's vector.
* **No metadata**: You cannot filter by product, date, or tenant, so every query searches the whole world. `source`, `as_of`, and `acl` are the unglamorous half of the pipeline. An organization-wide ingest still leaves the agent with whatever chunks you stored. Garbage chunks in, confident nonsense out.
* **`as_of` is two writes**: It is attached when the chunk is ingested. It is applied when the query is filtered. Write it and never filter, and the stale paragraph stays eligible. Filter a field you never stored, and the filter matches nothing. A weekly handbook needs both, or last month's policy remains the nearest neighbor.

Classic RAG retrieves once, then generates. An agent can call search, read the observation, reformulate, search again, or switch from vectors to keywords. That is a RAG agent, and it is also how retrieval loops run away. Budget the tool. Get search and relevance working, then wrap the agent, then add a second search style.

## 4. Semantic search and the index contract

**Indexing** rewrites a document so a later query can find it. Phrase match, keyword match, and meaning match are different indexes, even inside one product. How you will query is part of how you should index.

**Semantic search** ranks by meaning. You do not keep a synonym list so that "bike" finds "cycle." The embedding space is supposed to place them near each other. Demos feel like magic for that reason. Production fails in ways you can list in advance.

1. **Similarity is not truth**: The nearest chunk to "best treatment for a seized lock" may be a blog that sounds like a treatment and is wrong, expired, or from a tenant you must not search. The index knows nearby. It does not know correct. Use it to propose candidates. It breaks as a source of authority the moment you skip a filter and a grounding check.
2. **Top-k is a cliff**: With `k` of 5, the paragraph at rank 6 does not exist for the model. A question that needs hours and fees and exceptions wants a larger `k`, a split into sub-queries, or a second search. Raising `k` forever dumps the window. Splitting the question spends another hop.
3. **The user's words are not the manual's words**: "The thing that beeps when the dock is full" versus "overcapacity alarm (SPK-441)." Meaning sometimes bridges the gap. Exact codes often do not. Dropping keyword search because it sounds old misses `SPK-441`.

Write the contract before you pick a vendor. Two teams sharing an index without these answers have a pile. The agent still has to pass the right collection or tool. The wrong index is a silent access bug.

1. **Document**: A file, a page, a ticket, or a message. If two teams disagree, they will chunk different objects into one collection.
2. **Chunk**: Token budget, heading path, and overlap. This is the unit that gets a vector.
3. **Filterable metadata**: The fields a query is allowed to constrain. If it is not in the contract, the agent cannot ask for it.
4. **Embedder**: Model name, dimension, and similarity function. Pinned in the index metadata. An unnamed "the vendor's embedding" will change size under you.
5. **Re-embed**: On a document edit, on an embedder change, or on both. A new embedder without a re-index searches a space the old vectors do not occupy.
6. **Who may query**: Tenant, collection, and the tool that is allowed to see it. Isolated indexes do nothing if the agent passes the wrong name.

## 5. Lexical vectors, dense vectors, and cosine

Keep a method that does not capture meaning. You will still want it for tokens that must match exactly.

**TF-IDF**, and **BM25** in the ranking world, build a sparse vector. Most dimensions are words that are absent. A term scores high when it is frequent in this text and rare across the corpus. That is a claim about distinctive words, not about concepts.

```
query "vehicles"     doc "the cars in bay 3"
  TF-IDF misses (no shared term)
  dense can hit, if the embedder learned the nearness

query "SPK-441"      doc "SPK-441 dock overcapacity alarm"
  TF-IDF or BM25 hits the exact token
  dense may miss, or bury the code under essays about alarms
```

* **Lexical search**: A precision tool for tokens. Fast, and you can explain the hit. It wins on SKUs, names, and codes. It misses paraphrases, as "vehicles" misses "cars." Treating TF-IDF as a failed embedding throws away the instrument you need for `SPK-441`.
* **Dense embeddings**: A model trained so texts a person would call "the same topic" land near each other. You do not hand-write the features. You accept the model's biases, and you query with the same model you indexed with, or a documented compatible one.
* **One embedder per collection**: Mixing two models in one index compares vectors that were not trained to share a space. Pin the model name in the index metadata. One vendor's name is not one vector size forever.
* **Short queries are weak**: "hours?" and a long chunk share a space only because the trainer built it that way. Expand the query with HyDE, a rewrite, or a keyword fallback. Multilingual text and code need embedders that actually saw those domains.

**Cosine similarity** measures how aligned two directions are, and ignores length. Near 1, the directions match. Near 0, this embedder treats them as unrelated. Near -1, they oppose. Opposition is rare in trained text spaces, and the math still allows it.

* **Distance sorts the other way**: Some stores return distance. A common conversion is `cosine_distance = 1 - cosine_similarity`. Under that convention, 0 is the nearest neighbor. Sort descending and the top 5 are the farthest vectors you asked for. The bug looks like "embeddings do not work."
* **Read the store's unit**: Chroma-style results often surface distance. Port the same numbers to another store and `0.12` may be a near hit or a bad one. A toy of sparse vectors, cosine, and top-n is worth running once so the database feels like persistence. Ship the store that upserts, filters, and survives a restart.

## 6. A vector store

Once you have vectors, you need nearest neighbors, metadata filters, and a corpus that is not re-embedded every time the process starts. Naming a hosted vendor before naming the interface skips the work.

The interface is small: add embeddings, query by vector (and by text if the store embeds for you), filter on metadata, persist. The agent calls `search`. It should not care which implementation sits behind the tool.

1. **Embedded or local**: Chroma, LanceDB, or a SQLite extension. A client, a collection, `add`, and `query` make the lifecycle visible. Right for a workshop and for a single agent process. Painful when you need several processes, an SLA, or an access-control story you have not built.
2. **Postgres with a vector type**: One database you already operate. pgvector is the common direction. Right when the rest of the data already lives there. Awkward when the vector workload needs features this extension does not have.
3. **A dedicated service**: Qdrant, Weaviate, Pinecone, and others. You are buying operations, an SLA, and whatever hybrid features that product actually ships. Wrong as a first sentence if you still cannot describe add, query, filter, and delete.

The query loop is the same shape on each of them.

1. Embed the query with the ingest embedder, unless the store embeds for you and that embedder is the one you configured.
2. Ask for the top `n`, optionally filtered. Examples: `source = "spoke-ops"`, or `as_of` at or after a date.
3. Read the documents, the distances or scores, the metadata, and the ids.
4. Decide. Enough, reformulate, fall back to keywords, or refuse.

* **Empty is a result**: The persona says the handbook does not contain it. Free association on zero hits invents policy.
* **Ids are how you cite and delete**: De-duplicate, cite, and withdraw a document by id. "The third chunk in this response" cannot drive a deletion job when legal pulls a paragraph.
The store answers nearest-neighbor. The rest of the system is still yours.

* **Truth**: A near chunk can be wrong. Cosine does not check it.
* **Honesty of the generator**: The store cannot stop the model from ignoring the chunk. That is a grounding check.
* **Chunking**: PDFs are split before they are embedded. The database does not do that job for you.
* **Access**: An ACL works only if the field is in the metadata and the query filters on it.
* **Withdrawal**: When legal pulls a paragraph, something you run has to delete or re-index it. A missing delete is not cosine's fault.

## 7. Relevance, then a vector agent

Nearest neighbor is a foundation. Twelve "close" chunks, and a hope that the generator will sort truth from rhyme, is a retrieval failure dressed up as a prompt problem. A chunk belongs in the prompt when a careful reader could answer from it, cite it, and not be contradicted by its neighbor. If you cannot say why it is there, it should not be there.

| Failure | What you see | Move |
|---|---|---|
| Exact token miss | Query `SPK-441`, hits are essays on alarms | Keyword, BM25, or hybrid |
| Jargon mismatch | "Beep dock" versus "overcapacity" | Semantic search, a synonym list, or hybrid |
| Near-duplicates | Twelve copies of one spec | Dedup by id or hash; a diversity pass in the style of MMR |
| Stale chunk | The old paragraph is still nearest | `as_of`, re-ingest, a time filter |
| Wrong corpus | An HR index answers an ops question | Separate collections, and route the tool |
| Rank cliff | The answer sat at position 8 and `k` is 5 | Raise `k`, split the question, or search again |
| Thin or empty hits | The store has nothing useful | Refuse. Do not improvise the policy |

Build the vector agent before the hybrid one, so you have a baseline.

1. Load a corpus you control. A workshop can use one long text: an ops manual, a script, a catalog.
2. Chunk on a token budget, with a little overlap.
3. Embed, and upsert into a local collection.
4. Expose one tool, such as `search_handbook(query)`, returning passages.
5. Tell the persona to answer from passages, to say when the passages do not contain the answer, and to cite chunk ids when it can.

That loop already senses the question, forms a query, calls the tool, and reads the hits, including a possible second search. A clear tool name and docstring beat a 2,000-word order to "always call search." Then measure whether the model calls it.

* **A constraint is not a checker**: "Only claims supported by hits" belongs in the instructions. A later check is a different component. The constrained persona still invents. The check is how you catch it.
* **Probe, empty collection**: The tool returns no hits. The agent has to refuse. Skip this case and you never learn whether "I don't know" is implemented.
* **Probe, one obvious fact**: The answer has to cite it, including a chunk id when you have one.
* **Probe, a paraphrase**: "Beep when the dock is full" should still find "overcapacity" on the vector path.
* **Probe, an exact code**: `SPK-441` will often miss on vectors alone. That miss is the ticket for hybrid search. Note it. Do not "fix" it by stuffing the code into the persona.

## 8. Hybrid search

Run a keyword list and a vector list on the same query. The scores are not on one scale. BM25 and cosine cannot be sorted together raw. **Fusion** builds one list.

**Reciprocal rank fusion (RRF)** ignores the raw scores and uses rank. A typical fused score is the sum, over lists, of `1 / (k_rrf + rank)`, with a small constant `k_rrf`. 60 is a common default. An item high on both lists rises. An item that wins only one list still survives, which is the point: the `SPK-441` hit is not thrown out because its embedding was mediocre.

```
query
  |-- keyword list    SPK-441 at rank 1
  |-- vector list     a "dock full" essay at rank 1
           |
           v
        RRF merge --> shared top chunks --> generator
```

1. **One hybrid tool**: The store, or a wrapper, fuses before the model sees the lists. The model stays simple, and you tune fusion offline. If a shared endpoint already fuses, call that. Rebuilding RRF inside the persona duplicates a knob you can no longer see.
2. **Two tools**: `search_lexical` and `search_semantic`. The agent picks, or calls both and merges in prose. You gain a trace of the choice. You also gain a way to skip the list that would have hit.
3. **A chain**: Keywords first, then vectors if the keyword list is empty, or the reverse. You can test the policy, and you can read it. It is not automatically better than RRF. Interpretability is the reason to choose it. Hit rate is the reason to keep RRF.

Hold out two kinds of question before you trust a fusion constant copied from a post.

* **Exact token**: `SPK-441` has to appear in the top ranks. Keyword-only should already hit. Vector-only may bury it.
* **Paraphrase**: "the dock that will not take more bikes" has to find the overcapacity alarm. Vector-only should already hit. Keyword-only may miss.
* **One knob**: Compare hit rate at rank 5 for vector-only, keyword-only, and fused. Change `k_rrf`, or the chain order, and read the table again. A constant nobody measured will quietly miss one of those two classes.

## 9. Memory writes and graph memory

Document RAG and memory RAG both retrieve. They do not share writes, identity, or deletion. **MCP** is the socket that exposes a store as tools, so the agent binary does not grow a Chroma client or a graph client per store. Community servers exist for graphs, vectors, and files. A socket is not a retention policy.

The cognitive labels are a filing cabinet. They are not a claim that the agent has a human memory system. Name the pieces you run.

| Piece | What it actually is |
|---|---|
| Context window | Tokens in this call. Short-term or working memory. It dies when you truncate. |
| External store | Survives the process. Long-term memory. MCP is the socket in front of this store, not a replacement for the window. |
| Scratch or state | The plan, notes, and tool traces for the current task. |
| Retrieval | The query that copies a slice back into the window. |
| Write policy | What is stored, under which id, and with whose consent. |
| Forgetting | TTL, summary, deletion, and what to do when a new fact contradicts an old one. |

* **Two write paths**: "Add memory" often becomes a second vector index of raw chat, queried like the handbook. A knowledge write ingests a document. A memory write records that this user is vegetarian, or that last Tuesday's refund needed a manager. Different schema, different tool, different retention.
* **No write, no long-term memory**: If the process never stores a row, next weekend's session cannot retrieve the preference. If it stores every turn, including every "ok," the store is a landfill and retrieval gets worse.
* **What "remember" has to do**: Persist across sessions, retrieve what is relevant now, splice a short block into the prompt, and skip what is expired or belongs to someone else.

**Graph memory** is for relations. Vectors are weak at "who owns the autoclave, and what broke last time?"

```
(Maya:Person) --works_on--> (Kite:Project)
(Kite:Project) --uses--> (Autoclave-3:Equipment)
(Autoclave-3) --observation--> "seal failed 2026-08-12"
```

* **Query the neighborhood**: Ask by entity, by type, or by relation, then pull the neighboring facts. Nearest-paragraph search can return a page that mentions an autoclave and never states who owns this one. The neighborhood query walks `works_on` and `uses` and lands on the observation.
* **Graph RAG on a corpus**: Extract entities while ingesting, retrieve a subgraph, then generate. On memory, the same shape records people, constraints, and events from the conversation.
* **Tools, not a second brain**: Search nodes, add an observation, find related. A sequential-thinking pad stores the plan for this job. A graph stores world state that should still be true tomorrow. One server for both jobs, with no name for the difference, mixes a draft plan with a fact.
* **Identity, or the graph rots**: An extraction on every message that never merges "Maya", "maya@lab", and "Dr. Chen" mints a node per string. Use stable ids. Attach observations to the node, not a new node per synonym. Clean up on a schedule. A bad edge looks like knowledge, so this pile rots faster than a messy vector index.
* **Worth the graph**: Multi-hop questions, such as equipment on Maya's project that failed. Constraints that are relations, such as "do not email legal without cc'ing Maya." Inventories and org charts.
* **Wasted graph**: A single FAQ with no entities worth naming. A team that will not maintain extraction quality. The edges will look like answers and will be wrong.

## 10. Semantic, episodic, and procedural rows

Hybrid search fused two rankings of documents. **Hybrid memory** gives the agent two tools over two shapes: a vector store for prose ("what we talked about") and a graph or SQL store for facts ("who, what, and bound to whom").

A turn can run in a fixed order you can see in a trace.

1. Search semantic memory for conversation and notes.
2. Search the graph for entities and constraints.
3. Synthesize a short remembered block. Cap how many memories you inject.
4. Answer from that block plus the live tools.
5. Write back a new fact, a new episode, or nothing. Write after you know what was true. Every turn is usually the wrong cadence.

Instructions can describe that ritual, and they drift. Boring tool names and a fixed order are easier to trace. A required prefix such as "Remembering..." is a string the model can skip. A tool call is a span. Each store can be its own MCP server. You still decide what may be written (personal data, secrets), when the write happens, and how a conflict resolves when the user says they eat fish now.

Long-term rows split by what is stored. Give each row a handle so vector search can find it.

1. **Semantic**: Durable facts and meanings. "Cedar Ridge closes Mondays." "The user prefers an aisle." A vector or a key-value row. When a new fact contradicts it, invalidate the old one. Leaving both live means the prompt can quote either.
2. **Episodic**: An event. "On 12 Aug the user asked to move the reservation after the strike notice." Store a timestamp, the actors, and the outcome, plus a prose summary you can embed. A pure vector is hard to correct later ("delete the fee waiver"). A pure table is hard to find from a vague sentence.
3. **Procedural**: How this agent already does a thing. "To refund a Spoke day-pass, pull the trip id, then call `credit_wallet`." Steps, preconditions, and the last success. A playbook table, with the description embedded so a later query can find the playbook.

**Semantic augmentation** asks a model for the questions that should retrieve this episode or procedure later. Store those questions, or their embeddings, beside the record. Retrieval then matches how people ask, not how the log happened to be worded.

```
episode: "strike, moved reservation, waived fee"
        |
        v
model proposes trigger questions
        |
        v
embed questions + summary  -->  vector side
keep structured fields      -->  table or graph
```

* **Invent queries, not facts**: The model may write "what happens to the fee if a strike moves the trip?" It may not write a fee waiver that the episode does not contain. The record stays the source of truth.
* **Why the row is hybrid**: A vector-only row is hard to correct ("delete the fee waiver"). A table-only row is hard to find from a vague sentence. You pay design time. You avoid both failure modes.

## 11. Compression and forgetting

Stores fill with duplicates, stale preferences, and every "ok." **Compress** what you keep. **Forget** what you should not retrieve. Those are jobs, not metaphors.

Compression clusters near memories, by embeddings and k-means, or by a shared entity id. Summarize the cluster into one record. Keep pointers to the members if you still need an audit. A month of scheduling chatter becomes "the user usually cannot do Tuesdays."

* **Same entity, one cluster**: Three notes that all name Tuesday availability should collapse by entity id when you have one. k-means is the fallback when you only have embeddings and no stable id.
* **The compact row is for search**: It is what you inject into the window. It is not the record you authorize a refund from. The members, or their ids, stay available for an audit.

```
many near-duplicate memories
        |
     cluster
        |
   summarize --> one compact memory
        |
   optional: archive or drop the members
```

Forgetting is a policy you can name.

* **TTL**: Session scratch dies in hours. A dietary fact does not.
* **Frequency**: A procedural tip nobody uses should decay. One that runs daily stays.
* **Supersede**: The new fact replaces the old one. Retrieving both reopens the contradiction.
* **User deletion**: A requirement you can execute by id, not a later project.
* **Never store**: Secrets, raw card numbers, and someone else's data that arrived in a paste.

### Watch for

* A summary can become a false memory. Three chats about this month's strike become "the user hates Tuesdays," and the compact row has no episode id to check. Keep the source id.
* Operational fields want a structured replace, not lossy prose. Summaries are for search and for the window. They are not an authorization source.
* Truncating the live transcript is a budget for this call. Forgetting in the long-term store is how you stop the archive from growing mold. They are different jobs.

## Check yourself

1. Someone says a million-token window removes the need to retrieve. Which problems remain even when the whole handbook fits in the call?
2. A handbook changes weekly. Where is `as_of` written, and where is it filtered? What fails if you only do one of those?
3. Why are the embedding model and the generator two jobs? What breaks if you change the embedder and do not re-index?
4. For the query `SPK-441`, and for "the dock that will not accept more bikes," which family do you run first? What does rank fusion buy that picking one family does not?
5. A store returns cosine distance. You sort descending and keep five. What did you keep, in plain language?
6. The user says "I'm vegetarian." A cafeteria PDF lists Monday menus. Which store is written, which is read for "what can I eat with you today?", and what goes wrong if both rows share one collection with no type?
7. "Who owns autoclave 3, and what failed last?" Why is a neighborhood query a better interface than nearest-paragraph search, and when is building the graph wasted?
8. For a preference that must survive the weekend, what do the window, the external store, the scratch pad, and retrieval each hold? Which of those is MCP a socket for?
9. You augment an episode with a model. What may the model invent, what must stay literal, and why?
10. Compression turned three chats about this month's strike into "the user hates Tuesdays." Which provenance or forgetting rule would have stopped that compact row from living on as a fact?
