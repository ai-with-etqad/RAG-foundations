## 🔍 6. Retrieval Techniques (This is where pros stand out)

In mature RAG systems, embeddings alone do not create strong answers. Retrieval strategy is what determines whether the model receives the right evidence at the right time under real-world constraints such as latency, noisy queries, domain jargon, and multi-tenant filtering.

This chapter focuses on the retrieval techniques that differentiate basic demos from production-grade systems. These methods are not mutually exclusive. High-performing pipelines often combine several of them in staged retrieval and reranking flows.

### Top-K Retrieval

Top-k retrieval is the foundational retrieval pattern: return the `k` highest-scoring chunks for a query according to a similarity function (cosine similarity, dot product, Euclidean distance, BM25 score, or fused hybrid score).

Why it matters:

- It defines the initial candidate context seen by the LLM.
- It controls recall and context budget tradeoffs.
- It sets a baseline for all advanced retrieval improvements.

How it works conceptually:

1. Convert query to a retrieval representation (embedding and/or lexical query).
2. Search the index and rank candidates by score.
3. Return the top `k` candidates to downstream stages (direct prompt, reranker, compressor, or generator).

Key tuning dimensions:

- **Small `k` (for example 3-5):** lower latency and token use, but higher risk of missing relevant evidence.
- **Larger `k` (for example 20-100):** better recall potential, but noisier context and higher cost.
- **Dynamic `k`:** adapt `k` by query ambiguity, confidence, or user intent (fact lookup vs exploratory question).

Common failure modes:

- Important evidence ranked just below the cutoff.
- Redundant chunks consume top positions.
- High semantic similarity but low answer utility (chunk "about" topic but not containing needed fact).

Production best practices:

- Evaluate top-k with recall@k and answer accuracy, not score curves alone.
- Use top-k as a candidate generation step, then rerank/compress before final prompting.
- Tune separately per query class (short keyword query, long natural-language query, entity-heavy query).

### Similarity Thresholding

Similarity thresholding applies a minimum score requirement so low-confidence matches are excluded, even if they appear in top-k. This prevents forcing irrelevant context into prompts when retrieval confidence is weak.

Core intuition:

- Top-k always returns `k` results, even if all are poor.
- Thresholding allows returning fewer than `k` or even zero results when confidence is low.

Why this is critical:

- Reduces hallucination amplification from irrelevant chunks.
- Improves trustworthiness when retrieval confidence is uncertain.
- Enables fallback logic (clarifying question, query rewrite, web/tool search, "insufficient evidence" response).

Design patterns:

- **Static threshold:** single global cutoff (simple, but often brittle across domains).
- **Per-index threshold:** different cutoff by source type or embedding model.
- **Adaptive threshold:** calibrate based on query length, entropy, hybrid score distribution, or user intent.

Operational caveats:

- Raw similarity scores are not always calibrated across models/indexes.
- Threshold too high can harm recall for valid but subtle matches.
- Threshold too low behaves like plain top-k and may pass noise.

Practical implementation strategy:

1. Log score distributions for successful vs failed retrieval.
2. Tune thresholds with offline eval sets and online feedback.
3. Add fallback behavior for low-confidence retrieval (ask clarification, widen retrieval, or change retrieval mode).

### Maximal Marginal Relevance (MMR)

Maximal Marginal Relevance balances **relevance** and **diversity** during candidate selection. Instead of taking the top-ranked items directly, MMR penalizes redundancy so returned context spans different but relevant evidence.

Why MMR helps in RAG:

- Retrieval results often contain near-duplicate chunks from the same document region.
- Redundant context wastes tokens and reduces coverage of complementary evidence.
- Diverse retrieval improves robustness for multi-facet questions.

MMR objective (high level):

- Select items that are similar to the query.
- Penalize items too similar to already selected items.
- Control tradeoff with a diversity parameter (often `lambda`).

Tuning behavior:

- **Higher relevance weight:** more like classic top-k, less diversity.
- **Higher diversity weight:** broader coverage, possible drop in direct relevance.

Best-fit scenarios:

- Long documents where adjacent chunks are semantically overlapping.
- Multi-part questions ("compare A and B and mention risks").
- Knowledge bases with repetitive templated text.

Common pitfall:

- Over-diversification can include peripheral chunks that are distinct but not useful.

Good practice:

- Apply MMR to a moderately large candidate pool (for example top-30) and output a smaller final set (for example 6-10).
- Combine with reranking to recover precision after diversity selection.

### Hybrid Retrieval

Hybrid retrieval combines lexical and semantic retrieval signals so systems can capture both exact term matches and conceptual similarity.

Why hybrid retrieval is often superior:

- Lexical search excels at exact tokens (IDs, error codes, API names, legal phrases).
- Embedding search excels at paraphrases and semantic intent.
- Real user queries frequently contain both patterns.

Typical hybrid architecture:

1. Run BM25/keyword retrieval and vector retrieval in parallel.
2. Merge candidates.
3. Fuse rankings/scores (weighted blending, RRF, learning-to-rank).
4. Pass fused list to reranker/compressor/generator.

Score fusion approaches:

- **Weighted score fusion:** normalize and combine lexical + vector scores.
- **Reciprocal Rank Fusion (RRF):** robust rank-level merging without heavy score calibration.
- **Two-stage fusion:** candidate union first, cross-encoder reranker second.

#### BM25 + Embeddings

This is the most common hybrid baseline and frequently a strong production default.

BM25 contribution:

- Captures exact word overlap and term rarity.
- Strong for identifiers, abbreviations, and rare terms.

Embeddings contribution:

- Captures semantic relatedness and paraphrases.
- Strong when wording differs from indexed text.

Why combining both works:

- BM25 catches precision-critical exact signals.
- Embeddings recover relevant content when lexical overlap is low.

Implementation tips:

- Normalize scores before blending to prevent one retriever from dominating.
- Use separate top-k from each retriever, then fuse.
- Start with RRF as a stable baseline, then tune weighted fusion.

#### Multi-Query Retrieval (Generate Multiple Queries)

Multi-query retrieval expands one user query into multiple alternative formulations, then retrieves with each and merges results. It improves recall when a single phrasing underspecifies intent.

Why it works:

- User questions are often incomplete, ambiguous, or phrased narrowly.
- Alternative phrasings hit different vocabulary and semantic neighborhoods.
- Union of candidates reduces single-query blind spots.

Typical generation patterns:

- Paraphrases of original question.
- Sub-questions for multi-hop intent.
- Synonym/domain-term variants.
- Explicit entity-focused and definition-focused rewrites.

Pipeline:

1. Generate `n` alternate queries (often 3-8).
2. Retrieve per query.
3. Deduplicate and fuse candidates.
4. Optionally rerank globally before context assembly.

Tradeoffs:

- Higher recall, but extra latency and cost.
- More retrieved noise if generated queries drift off intent.

Guardrails:

- Constrain query generation with intent-preserving prompts.
- Drop low-quality generated queries using confidence checks.
- Cache generated expansions for repeated user intents.

#### Query Rewriting / Expansion

Query rewriting transforms a user query into a retrieval-optimized query. Query expansion adds relevant terms/entities without changing original intent. Both improve retriever compatibility.

When needed:

- Conversational follow-up questions lack context ("What about its latency?").
- User language is vague while corpus language is technical.
- Domain terminology mismatch (layperson wording vs specialist docs).

Rewriting examples:

- Add missing entities from session context.
- Convert pronouns to explicit references.
- Standardize time ranges, versions, or product names.

Expansion examples:

- Add synonyms and acronyms.
- Add adjacent terms likely needed for retrieval.
- Add canonical entity names and aliases.

Risks:

- Over-expansion introduces unrelated terms and noisy retrieval.
- Aggressive rewriting may distort user intent.

Best practice:

- Keep original query as anchor signal in fusion.
- Track rewrite provenance for debugging.
- Evaluate rewrite impact on both retrieval metrics and final answer quality.

#### Self-Querying Retrievers

Self-querying retrievers use an LLM to translate natural-language intent into a structured retrieval query that includes both semantic intent and metadata filters.

What makes them powerful:

- Users rarely express filters in strict query syntax.
- LLM can infer constraints like date ranges, document type, language, or product family.
- Retrieval becomes both semantically relevant and structurally constrained.

Example transformation pattern:

- User: "Recent incident reports about payment retries in Europe."
- Structured retrieval intent:
  - Semantic query: "payment retry incidents"
  - Filters: `region = Europe`, `doc_type = incident_report`, `created_at = last 90 days`

Advantages:

- Better precision for filtered enterprise corpora.
- More natural user interface for complex retrieval constraints.
- Reduced manual query-language burden on users.

Failure risks:

- Incorrect inferred filters can silently hide relevant documents.
- Metadata schema drift can break generated structured queries.

Operational safeguards:

- Validate generated filters against known schema.
- Log structured query plans for inspection.
- Provide fallback to unfiltered or relaxed retrieval when filter confidence is low.

#### Parent-Child Retrieval

Parent-child retrieval stores fine-grained child chunks for precise matching but returns larger parent contexts for generation. This combines retrieval precision with answerable context windows.

Why it matters:

- Small chunks improve matching granularity.
- LLMs often need broader surrounding context for coherent answers.
- Directly returning only tiny chunks can miss critical qualifiers.

How it works:

1. Split documents into child chunks (retrieval units).
2. Link each child to a parent section/document.
3. Retrieve top child chunks.
4. Promote to corresponding parents (or expanded windows) for final context.

Benefits:

- Better match quality than large-chunk retrieval.
- Better generation fidelity than tiny-chunk-only prompting.
- Useful for policy docs, technical manuals, and legal corpora.

Design considerations:

- Parent size should fit downstream context budget.
- Merge multiple child hits from same parent to avoid duplication.
- Preserve citation traceability from parent context back to matched children.

#### Context Compression

Context compression reduces retrieved text volume while preserving answer-relevant evidence. It is especially important when retrieval returns large passages but model context windows and latency budgets are limited.

Compression strategies:

- **Extractive compression:** keep only salient sentences/spans from retrieved chunks.
- **Query-focused summarization:** summarize candidate text around query intent.
- **Rerank + trim:** keep top evidence segments and drop weak tails.
- **Structured compression:** extract fields/facts into compact schemas.

Why compression improves systems:

- Lowers token cost and latency.
- Reduces distraction from irrelevant details.
- Increases signal density in final prompt context.

Key risk:

- Over-compression can remove qualifiers, negations, or edge-case details and lead to wrong answers.

Mitigation:

- Keep citations to original chunks for traceability.
- Apply conservative compression for high-stakes domains.
- Validate compressed-context answer accuracy against non-compressed baseline.

### Retrieval Strategy Composition (How pros combine techniques)

Strong systems rarely rely on one retrieval method. They layer techniques to improve recall, precision, diversity, and efficiency together.

A common production sequence:

1. Rewrite/expand query from conversation state.
2. Run hybrid retrieval (BM25 + embeddings) with moderate top-k.
3. Apply multi-query expansion for hard or ambiguous queries.
4. Use MMR or reranker to reduce redundancy and improve coverage.
5. Apply similarity thresholding and confidence-based fallback.
6. Use parent-child promotion and context compression before generation.

This staged design gives better control than any single retrieval trick. Retrieval quality becomes a system-level property shaped by query understanding, candidate generation, ranking, filtering, and final context assembly.

### Practical Evaluation Checklist

To improve retrieval techniques reliably, evaluate each layer with explicit metrics:

- **Recall-oriented:** recall@k, hit rate, coverage of gold evidence.
- **Ranking-oriented:** MRR/NDCG, reranker lift.
- **End-to-end:** answer correctness, groundedness/citation fidelity, user satisfaction.
- **Operational:** latency, token cost, failure fallback rate, drift over time.

A retrieval pipeline is "professional grade" when it is not only accurate in demos, but measurable, debuggable, and stable under changing queries, data, and production constraints.
