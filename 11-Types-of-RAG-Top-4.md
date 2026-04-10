## 🧩 11. Types of RAG (Top 4)

As RAG systems evolve from prototypes to production, one major realization appears quickly:

there is no single "best" RAG architecture for all problems.

Different query patterns, data distributions, and latency/cost constraints push teams toward different RAG types.  
This chapter covers the four most practical and widely used patterns:

1. Basic / Naive RAG.
2. Hybrid RAG (keyword + vector).
3. Multi-Hop / Advanced Retrieval RAG.
4. Graph RAG (knowledge graph-based).

The goal is to understand what each type is, how it works, when to use it, and where it can fail.

---

### 1. Basic / Naive RAG

Basic RAG is the canonical starting point and the fastest way to operationalize grounding.

#### Definition

Basic (or naive) RAG is a retrieval-generation pipeline where:

1. User query is embedded.
2. Similar chunks are retrieved from a vector index.
3. Retrieved chunks are appended to prompt context.
4. LLM generates an answer using those chunks.

The architecture is "naive" not because it is bad, but because it assumes a single-pass retrieval route is enough for most questions.

#### Pipeline Flow

A standard basic RAG flow looks like this:

1. **Ingestion:** collect docs (PDFs, webpages, docs, FAQs).
2. **Chunking:** split into fixed-size or simple semantic chunks.
3. **Embedding:** convert chunks into dense vectors.
4. **Indexing:** store vectors + metadata in vector DB.
5. **Query embedding:** convert user question into vector.
6. **Top-k retrieval:** fetch nearest chunks.
7. **Prompt assembly:** include instruction + retrieved context + question.
8. **Generation:** LLM returns grounded answer.
9. **(Optional) citation display:** show source snippets.

This flow is simple, explainable, and easy to ship in an MVP.

#### When to Use

Basic RAG is ideal when:

- Domain is narrow and documents are relatively clean.
- Most user questions are single-hop ("find and summarize" style).
- You need quick time-to-market with limited engineering overhead.
- Precision requirements are moderate (internal tools, assistant drafts).
- Corpus size is manageable and metadata complexity is low.

It is often the right first production milestone before adding complex retrieval orchestration.

#### Limitations

Basic RAG breaks down in several common scenarios:

1. **Lexical mismatch:** vector similarity may miss exact keyword/ID lookup needs.
2. **Single-hop bias:** it struggles when answer requires joining facts across multiple documents.
3. **Context dilution:** top-k chunks may include weakly relevant noise.
4. **No retrieval adaptation:** same strategy is used regardless of query type.
5. **Limited explainability:** if chunk selection is weak, answers degrade even when source truth exists.

Operationally, naive RAG can appear good in demos but fragile under diverse real-world query traffic.

#### Example Use Cases

- Internal HR policy assistant over handbook and SOP docs.
- Customer support bot over product documentation.
- Academic FAQ helper over course notes.
- Startup knowledge bot over Notion/wiki pages.
- First-version legal clause explainer (low-risk tier).

---

### 2. Hybrid RAG (Keyword + Vector)

Hybrid RAG combines lexical precision with semantic similarity to improve retrieval robustness.

#### Definition

Hybrid RAG retrieves context from two complementary signals:

1. **Keyword/lexical retrieval** (typically BM25 or inverted index).
2. **Dense vector retrieval** (embedding similarity).

Results are then fused (before or after reranking) into a single evidence set for generation.

The key idea: lexical and semantic retrieval fail differently, so combining both improves overall recall and precision across mixed query types.

#### BM25 vs Embeddings

Both methods solve relevance, but with different strengths.

**BM25 (keyword / lexical retrieval):**

- Excels at exact terms, numbers, IDs, acronyms, code names.
- Handles rare tokens and literal match constraints well.
- Interpretable scoring and stable for deterministic lookup tasks.
- Weak on paraphrases and semantic intent mismatch.

**Embeddings (dense retrieval):**

- Excels at semantic similarity and paraphrased intent.
- Better for natural language questions not using exact doc wording.
- Strong for concept-level matching across vocabulary variation.
- Can miss exact string constraints and may retrieve semantically close but wrong facts.

In practice, enterprise queries contain both patterns in the same session.  
That is why hybrid retrieval is often a default for production-grade assistants.

#### Fusion Strategies

Common fusion strategies include:

1. **Weighted score fusion:** normalize lexical and vector scores, then combine with tunable weights.
2. **Reciprocal rank fusion (RRF):** combine ranks from both retrievers; robust and simple.
3. **Cascade retrieval:** lexical first for exact constraints, then semantic expansion.
4. **Parallel retrieval + reranker:** fetch candidates from both channels, rerank with cross-encoder.
5. **Query-aware dynamic fusion:** adjust lexical/semantic weights by detected query intent.

A practical pattern is RRF + reranker because it is strong, interpretable, and easy to iterate.

#### When to Use

Hybrid RAG is preferred when:

- Queries mix natural language with precise identifiers.
- Domain includes SKUs, legal references, error codes, tickets, part numbers.
- Users phrase the same question with varied wording.
- Missing relevant evidence is more costly than slightly higher retrieval compute.
- You need a reliable baseline before advanced multi-hop orchestration.

#### Tradeoffs

Hybrid retrieval improves robustness, but introduces complexity:

1. More infrastructure (lexical + vector indices).
2. More tuning surface (weights, cutoffs, normalization, reranking).
3. Slightly higher latency due to multiple retrieval passes.
4. More observability needs (per-channel contribution tracking).
5. Potential over-recall if fusion is not calibrated (more noise chunks).

The additional complexity is usually justified for heterogeneous enterprise corpora.

#### Example Use Cases

- IT support assistant answering error code + troubleshooting intent.
- E-commerce assistant combining SKU lookup and semantic product Q&A.
- Compliance assistant handling regulation IDs plus policy interpretation.
- Developer docs assistant for API symbols and conceptual explanations.
- Healthcare ops assistant matching procedural codes and free-text guidance.

---

### 3. Multi-Hop / Advanced Retrieval RAG

Multi-hop RAG is built for questions where one retrieval pass is not enough to collect all required evidence.

#### Definition

Multi-hop (advanced retrieval) RAG performs retrieval as an iterative reasoning process rather than a single top-k fetch.

Instead of asking "what chunks are closest to this query once?", it asks:

1. What sub-questions must be answered?
2. What evidence is missing after first retrieval?
3. What follow-up retrieval should be run next?

This pattern is crucial for compositional questions spanning entities, timelines, and causal chains.

#### Multi-Hop Reasoning

Multi-hop reasoning means answer construction depends on combining evidence from multiple linked facts.

Example pattern:

- Hop 1: identify the relevant entity/project/policy.
- Hop 2: retrieve latest metric/status tied to that entity.
- Hop 3: retrieve explanatory context (incident notes, change logs, constraints).
- Final: synthesize a grounded answer across all hops.

Without multi-hop logic, systems often return a partial answer from the first matching chunk and miss the full causal or relational picture.

#### Query Decomposition

Query decomposition splits a complex user request into structured sub-queries.

Typical decomposition workflow:

1. Detect whether query is compositional (contains multiple intents/constraints).
2. Generate sub-questions with dependency ordering.
3. Retrieve evidence per sub-question.
4. Track unresolved slots (unknown entity, date window, condition).
5. Continue until required evidence set is complete.

High-quality decomposition significantly improves retrieval precision because each sub-query is narrower and better aligned with source structure.

#### Iterative Retrieval

Iterative retrieval closes evidence gaps over multiple rounds.

A robust loop:

1. Retrieve initial evidence.
2. Evaluate evidence sufficiency (is anything missing/ambiguous?).
3. Generate follow-up query from unresolved gaps.
4. Retrieve again from targeted sources.
5. Merge and deduplicate evidence.
6. Stop when confidence threshold or hop limit is reached.

Guardrails are important:

- Maximum hop count to avoid runaway loops.
- Confidence thresholds for early stopping.
- Contradiction checks across hops.
- Per-hop logging for observability and debugging.

#### When to Use

Use multi-hop RAG when:

- Questions require joins across documents/sources.
- Users ask "why/how" questions, not only "what".
- Domain requires timeline + entity + policy composition.
- First-pass retrieval often gives incomplete answers.
- High-stakes workflows require deeper evidence coverage.

#### Example Use Cases

- Root-cause analysis assistant over incident, deployment, and monitoring logs.
- Financial research assistant linking company events to metric shifts.
- Clinical ops assistant combining protocol, patient factors, and latest guidelines.
- Legal analysis assistant connecting clauses, precedents, and jurisdiction constraints.
- Enterprise analytics copilot for KPI explanation across multiple data artifacts.

---

### 4. Graph RAG (Knowledge Graph-based)

Graph RAG introduces explicit entity-relationship structure to improve relational retrieval and reasoning.

#### Definition

Graph RAG augments (or partially replaces) pure chunk retrieval with a knowledge graph where:

- Nodes represent entities (people, products, systems, concepts, events).
- Edges represent relations (owns, depends_on, causes, located_in, version_of, governed_by).

At query time, the system can traverse graph neighborhoods to collect connected evidence and then ground generation with both graph facts and supporting text.

#### Graph Construction (Entities & Relations)

Graph quality determines Graph RAG quality.

A typical graph construction pipeline:

1. **Entity extraction:** detect canonical entities from documents/tables.
2. **Entity resolution:** merge aliases and duplicates into unified IDs.
3. **Relation extraction:** identify typed edges between entities.
4. **Temporal/context tagging:** attach timestamps, source scope, confidence.
5. **Graph storage:** persist in graph DB (or graph layer over existing store).
6. **Back-linking to evidence:** keep provenance to original chunks/docs.

Best practice is to preserve confidence and provenance on each edge so downstream reasoning can stay auditable.

#### Graph Traversal vs Vector Search

Both methods can coexist, but they optimize different retrieval goals.

**Graph traversal:**

- Strong for relational questions ("how is X connected to Y?").
- Supports path-based reasoning and neighborhood expansion.
- Better for multi-entity dependency chains.
- Requires graph maintenance and schema discipline.

**Vector search:**

- Strong for semantic recall from unstructured text.
- Faster to deploy and simpler to maintain initially.
- Better for broad natural language matching.
- Weaker for precise multi-step relational constraints.

A common production pattern is **Graph + Vector hybrid**:

1. Use graph traversal to identify relevant entity subgraph.
2. Use vector retrieval to pull rich textual evidence for those entities.
3. Fuse both for grounded answer generation.

#### When to Use

Graph RAG is ideal when:

- Domain is relationship-heavy (org structure, supply chain, biomedical, fraud, legal networks).
- Questions frequently require "connection" or "dependency" reasoning.
- Entity identity consistency matters across many sources.
- You need explainable path-based evidence, not only nearest-neighbor chunks.
- Long-term knowledge curation is a strategic asset.

#### Tradeoffs

Graph RAG offers strong reasoning structure but costs more to build and operate:

1. Higher upfront modeling effort (ontology/schema design).
2. Ongoing maintenance for entity resolution and relation quality.
3. More complex ingestion and governance workflows.
4. Potential brittleness if graph extraction quality is low.
5. Extra infrastructure and specialized team knowledge.

For relationship-centric domains, these costs are often outweighed by better correctness and explainability.

#### Example Use Cases

- Fraud detection assistant over transaction and identity networks.
- Biomedical research assistant over gene-disease-drug relationships.
- Supply chain copilot for upstream/downstream dependency analysis.
- Enterprise architecture assistant mapping service dependencies and incidents.
- Legal intelligence assistant over entities, contracts, obligations, and precedents.

---

### Choosing the Right RAG Type

In practice, teams rarely stay with a single type forever.  
Most systems evolve along this path:

1. Start with **Basic RAG** for speed.
2. Move to **Hybrid RAG** for better retrieval robustness.
3. Add **Multi-Hop retrieval** when complex reasoning gaps appear.
4. Introduce **Graph RAG** when relational structure becomes core to product value.

The strongest production systems are often layered hybrids rather than pure forms of one type.

If you optimize for real workloads (not demo prompts), your architecture naturally converges toward query-aware retrieval orchestration with strong observability and evidence grounding.
