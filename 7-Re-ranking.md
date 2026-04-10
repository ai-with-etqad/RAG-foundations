## 🧠 7. Re-ranking (Huge performance boost)

Re-ranking is one of the highest-leverage improvements in production RAG systems. Initial retrieval (vector, lexical, or hybrid) is designed for speed and broad recall, which means it often returns a candidate set that contains useful evidence mixed with irrelevant or weakly relevant chunks.

A reranking stage fixes that quality gap by reordering the retrieved candidates using a stronger relevance model. The result is usually better context quality, better answer accuracy, and lower hallucination risk without rebuilding the entire retrieval stack.

In practical deployments, re-ranking is often the point where a "good demo" becomes a "reliable system."

### Why Initial Retrieval Is Often Noisy

Most first-stage retrievers are optimized for fast candidate generation, not deep semantic precision. This is an intentional design choice: if the first pass is too strict, recall collapses and important evidence is missed.

Why noise appears in the first stage:

- **Embedding similarity is broad:** Chunks can be semantically related to the query topic but still not answer the exact user intent.
- **Lexical retrieval is literal:** BM25 and keyword search can over-prioritize surface term overlap, even when the passage is contextually wrong.
- **Chunk granularity mismatch:** Chunks may be too large (mixed topics) or too small (insufficient context), causing unstable ranking quality.
- **Domain-specific ambiguity:** Terms can be overloaded (for example "index", "agent", "latency"), and first-pass ranking may select the wrong sense.
- **Multi-intent queries:** A single query can contain comparison, constraint, and reasoning requirements that first-stage models do not fully capture.

Typical symptoms of retrieval noise:

- Top results repeat similar information but miss the key fact needed for the answer.
- High-scoring chunks are "about" the topic but do not contain actionable evidence.
- Correct evidence appears lower in the list (for example rank 15), outside the final context window.
- Generated responses sound fluent but cite shallow or weak support.

Why this matters in RAG:

- The LLM can only reason over supplied context. If relevant evidence is buried below the cutoff, answer quality drops.
- Noisy context increases the chance of confident but wrong synthesis.
- Token budget gets wasted on low-value chunks, reducing room for truly useful evidence.

Core insight:

1. First-stage retrieval should maximize recall and speed.
2. Re-ranking should maximize precision on that candidate set.
3. Together, they create a strong retrieval pipeline with balanced quality and latency.

### Cross-Encoder Models for Reranking

Cross-encoders are the most common high-precision reranking models. Unlike bi-encoder retrieval (which embeds query and documents separately), a cross-encoder reads the query and candidate chunk together in one forward pass and outputs a relevance score.

Why cross-encoders are strong:

- They model fine-grained interactions between query tokens and passage tokens directly.
- They can capture nuanced intent such as negation, constraints, and specific entity relations.
- They generally outperform pure embedding similarity when ranking a small candidate set.

How cross-encoder reranking works:

1. Retrieve an initial candidate set (for example top-50 from vector/hybrid retrieval).
2. For each candidate, form a pair input: `[query] + [candidate chunk]`.
3. Run the pair through the cross-encoder to obtain a relevance score.
4. Sort candidates by cross-encoder score.
5. Keep final top-N chunks (for example 5-10) for prompt assembly.

Why this is usually a two-stage pattern:

- Cross-encoders are computationally expensive when applied to entire corpora.
- They are ideal for reranking a limited candidate pool, not for full index search.
- This preserves first-stage speed while adding second-stage precision.

Important implementation details:

- **Candidate pool size:** Too small and recall is capped; too large and latency/cost increases.
- **Chunk quality:** Better chunk boundaries improve reranker signal quality.
- **Model/domain fit:** General rerankers work well, but domain-adapted models can improve ranking in specialized corpora.
- **Truncation behavior:** Long chunks may get truncated; prioritize high-density, answer-bearing passages.

Common operational pattern:

- Stage 1: Hybrid retrieve top-30 to top-100 candidates.
- Stage 2: Cross-encoder rerank candidates.
- Stage 3: Optional compression/merging to reduce redundancy.
- Stage 4: Send top context to generator LLM.

### Learning-to-Rank Basics

Learning-to-rank (LTR) is a broader framework for training ranking systems to order candidates by expected relevance. Cross-encoders can be part of an LTR stack, but LTR also includes feature-based and ensemble ranking approaches.

LTR objective in RAG:

- Given a query and a set of candidate chunks, learn a scoring/ranking function that places the most useful evidence at the top.

Core LTR paradigms:

- **Pointwise:** Predict relevance score for each query-document pair independently.
- **Pairwise:** Learn which of two candidates should rank higher for a query.
- **Listwise:** Optimize quality over the entire ranked list directly.

In production retrieval, LTR can combine multiple signals:

- Vector similarity score
- BM25 or lexical score
- Cross-encoder relevance score
- Metadata priors (freshness, trusted source, document type)
- Behavioral signals (click-through, answer acceptance, user feedback)

Why LTR is powerful:

- It merges heterogeneous retrieval signals into one learned ranking policy.
- It adapts ranking behavior to your real workload and relevance labels.
- It can outperform single-signal ranking once enough high-quality data is available.

Evaluation metrics commonly used:

- **NDCG@k:** Rewards correct ordering near the top.
- **MRR:** Focuses on how quickly the first relevant result appears.
- **Recall@k:** Measures coverage of relevant evidence in top-k.
- **Precision@k:** Measures relevance density in final context window.

Data and labeling considerations:

- Human relevance labels are high quality but expensive.
- Weak labels from click logs are scalable but noisy.
- Continuous evaluation is required because query distributions drift over time.

Practical LTR adoption path:

1. Start with hybrid retrieval + cross-encoder rerank baseline.
2. Log rich features and outcomes.
3. Build relevance dataset from offline labels and online signals.
4. Train and evaluate LTR model against baseline.
5. Roll out gradually with A/B testing and guardrails.

### Tools: Cohere Rerank

Cohere Rerank is a managed API for high-quality document reranking. It is commonly used as a drop-in second-stage ranker after vector or hybrid candidate retrieval.

Why teams adopt it quickly:

- Fast integration with standard retrieval stacks.
- Strong out-of-the-box relevance quality for many domains.
- No need to host and serve your own reranker model infrastructure.

Typical usage pattern:

1. Retrieve top-k candidates from your primary retriever.
2. Send query + candidate documents to Cohere Rerank.
3. Receive reranked results with relevance scores.
4. Select top-N for prompt context.

Where it fits best:

- Teams that want quality gains without building custom reranker training pipelines.
- Applications where second-stage API latency is acceptable within SLA.
- Systems that need rapid iteration and measurable relevance lift.

Operational considerations:

- **Cost:** Charged per rerank call or tokenized workload (plan-dependent).
- **Latency:** Adds extra network + inference hop; candidate size matters.
- **Privacy/compliance:** Review data-handling policies before sending sensitive content.
- **Fallbacks:** Define behavior when API errors, rate limits, or timeouts occur.

Best practices with managed rerank APIs:

- Keep candidate pool reasonably sized (for example 20-80).
- Pre-filter obvious low-value chunks before rerank to reduce spend.
- Cache rerank outputs for repeated queries where feasible.
- Track rerank impact on answer correctness, not only rerank scores.

### Latency vs Accuracy Tradeoff

Re-ranking introduces one of the most important system-level tradeoffs in RAG: better relevance quality versus additional latency and compute cost.

Why the tradeoff exists:

- Higher-precision models (cross-encoders, stronger rerank APIs) require more computation.
- More candidates reranked increases potential recall/precision but linearly increases scoring work.
- Real-time products operate under strict user experience latency budgets.

Levers that control latency-accuracy balance:

- **Initial candidate count (`k1`):** Larger `k1` helps recall but increases rerank workload.
- **Final context count (`k2`):** Smaller `k2` reduces prompt tokens but risks missing supporting evidence.
- **Model size/quality:** Stronger models can improve ranking but are slower.
- **Batching and parallelism:** Improves throughput and can reduce effective per-query overhead.
- **Caching strategy:** Reduces repeated reranking for popular or repeated intents.

Common operating regimes:

- **Low-latency chat assistants:** Smaller candidate sets, aggressive caching, lightweight reranker.
- **High-stakes QA/compliance workflows:** Larger candidate sets, stronger reranker, stricter validation.
- **Asynchronous research workflows:** More generous latency budget for higher precision reranking.

A practical tuning loop:

1. Set clear SLA targets (for example p95 latency).
2. Benchmark multiple `(k1, reranker, k2)` configurations.
3. Measure end-to-end outcomes: answer quality, factuality, latency, and cost.
4. Pick the best Pareto point for your product context.
5. Re-tune periodically as corpus, query mix, and models evolve.

### Recommended Production Pattern

For most systems, the strongest default is a staged retrieval architecture:

1. **Candidate generation:** Hybrid retrieval optimized for recall and speed.
2. **Precision upgrade:** Cross-encoder or managed reranker on top candidates.
3. **Context shaping:** Deduplicate/compress and enforce token budget.
4. **Generation + verification:** Produce answer, optionally with citation validation.

This pattern provides consistent gains because each stage has a clear role:

- Stage 1 finds what might be relevant.
- Stage 2 decides what is actually most relevant.
- Stage 3 ensures context budget is used efficiently.

When teams skip reranking, they often compensate by increasing top-k and prompt size, which usually increases cost faster than it improves quality. Re-ranking is the cleaner and more controllable quality lift.
