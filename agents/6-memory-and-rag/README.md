# Working with memory and knowledge

A training cutoff is a wall. Yesterday's upload, last week's policy, and a preference from the previous session all sit on the other side. Retrieval searches a store that can grow on its own and places only the useful pieces into the one prompt, which is still finite. Knowledge is outside material: documents, tables, logs. Memory is the agent's own experience, fetched by the same move.

RAG has two phases. Ingestion loads documents, cuts chunks, embeds them, and writes the index. At question time you embed the query in that same space, pull the nearest chunks, and let a language model write from them. The embedding model and the answering model do different jobs.

Semantic search compares meanings, so a question about vehicles can land on a passage about cars. Three caveats ride along. A close neighbor can still be a false, stale, or untrusted passage. A fixed top-k quietly drops whatever sat at rank eleven. Approximate nearest-neighbor search trades a guarantee for speed.

Pick the store by the question. Prose wants vectors. A code, a price, or a join wants a table. "Who reports to whom" wants a graph. A practical agent carries more than one.

- **Embeddings and Chroma.** A dense vector is a few hundred to a few thousand floats, with related meanings nearby. Chroma, fine as a local store while you learn, ranks neighbors by cosine distance: zero is nearly the same text, and two points the other way. TF-IDF and BM25 still win when the exact error code, name, or rare token is the point.
- **Vector search and hybrid search.** A vector-only agent misses literal strings, mixes up jargon, lets duplicates crowd the top, cannot hop a relationship, and can serve a stale price. Run a keyword search beside the vector search and fuse the ranked lists. Reciprocal rank fusion scores by rank, so a keyword score and a cosine score never have to share a scale. Have the answer quote the passages, and say so when the source is silent.
- **Memory through MCP.** Point the same loop at the past. A graph memory server holds entities, relationships, and observations the agent chooses to write. A vector collection holds the wording of earlier talk. Each turn, search by meaning, walk the graph for the entities those hits named, and store anything new in both.
- **Semantic, episodic, and procedural memory.** Facts and concepts, events, and the steps of a process. Events and procedures sit naturally in rows. Embed them too, and draft the questions a later user might ask, so a future lookup has more ways in.
- **Compression and forgetting.** Cluster near-duplicates and replace each cluster with one summary. Compress when the items are cousins. Drop entries when the store is the same fact said again. Memory wants this on a schedule. A knowledge load usually wants it once. In healthcare, finance, and law, a retention rule can make eager deletion the larger risk.

The chapter's stores are learning props. They vanish when the process exits. Production still wants a durable backend, permissions on every lookup, and a check that the right memory came back.
