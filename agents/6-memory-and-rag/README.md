### **Chapter 6 Summary: Working with Memory and Knowledge**

Chapter 6 of *AI Agents in Action, Second Edition* covers retrieval-augmented generation, vector and hybrid search, practical knowledge agents, and MCP-backed memory architectures that extend agents beyond fixed model training.

---

#### **1. Understanding Retrieval in AI Applications**
* **Retrieval bridge**: **Retrieval** connects unbounded external stores to the **bounded context window** by fetching only the most relevant pieces for each LLM call.
* **Knowledge versus memory**: **Knowledge** is external reference material; **memory** captures experiential interaction history—both use the same **retrieve-then-augment** pattern.
* **RAG phases**: **RAG** splits into **ingestion** (load, chunk, embed, index) and **query-time retrieval** (embed query, search, augment prompt, generate).
* **Semantic search**: **Dense embeddings** compare meaning rather than surface keywords, enabling matches like “vehicles” to “cars,” with pitfalls around **factual correctness**, fixed **top-k**, and **approximate** nearest-neighbor indexes.
* **Why agents need retrieval**: Models cannot answer from **post-training facts** unless the system supplies current documents, policies, prices, or session-specific state at query time.

---

#### **2. Vector Databases and Similarity Search**
* **TF-IDF baseline**: **TF-IDF** builds sparse vectors from term importance and supports **cosine similarity**, but cannot align different words with the same concept.
* **Neural embeddings**: Pretrained **embedding models** map text into dense vectors where related meanings sit nearby, trained by predicting contextual structure rather than counting tokens.
* **Cosine similarity**: **Cosine similarity** compares vector direction and is standard for both sparse and dense retrieval, ignoring magnitude effects from document length.
* **In-memory and Chroma**: Vectors can live in arrays for experiments; **Chroma DB** provides a practical local **collection** API with distance-ranked query results.
* **Hybrid signal**: Production systems often blend **BM25/keyword** precision with **semantic recall** because neither alone covers codes, jargon, duplicates, and freshness.

---

#### **3. Building Practical RAG Knowledge Agents**
* **Vector-only limits**: A **vector-search RAG agent** grounded in ingested chunks answers well on semantic questions but may miss **exact times, codes, or rare strings** without complementary search.
* **Grounding**: **Grounding** instructions require answers to cite retrieved passages and admit when context is insufficient, reducing **hallucination** from parametric model knowledge.
* **Hybrid RAG agent**: Agents with separate **vector and keyword tools** choose or combine methods, enforcing quotes and reference markers in structured answer formats.
* **Relevance engineering**: Tables of failure modes—missed literals, jargon drift, duplicate crowding, stale facts—motivate **keyword**, **SQL**, **graph**, and **hybrid** supplements beyond pure similarity.
* **Reciprocal Rank Fusion**: **RRF** merges ranked lists from heterogeneous scorers by rank position, avoiding calibration problems between BM25 and cosine scores.

---

#### **4. Adding Memory to Agents with MCP**
* **Architectural memory view**: Agent memory decomposes into **context window**, **external storage**, **state scratchpads**, and **retrieval mechanisms** rather than purely cognitive metaphors.
* **Graph memory via MCP**: The MCP **memory server** exposes tools to store **entities, relationships, and observations** in a graph optimized for traversing connections.
* **Hybrid memory**: **Hybrid memory** runs **semantic search** first for conversational context, then **graph lookups** for structured facts, writing new information to both stores.
* **Semantic, episodic, procedural**: **Episodic** events and **procedural** steps fit relational patterns; **semantic augmentation** generates likely future queries to improve later recall of those records.
* **Compression and forgetting**: **Cluster-and-summarize compression** collapses near-duplicate memories; **forgetting** or eviction controls noise, with regulated domains balancing retention rules against pruning risk.

---
