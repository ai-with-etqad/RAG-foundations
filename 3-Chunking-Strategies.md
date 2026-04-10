## ✂️ 3. Chunking Strategies (SUPER important)

Bad chunking = bad RAG. Even with strong embedding models and hybrid retrieval, poorly segmented text causes the retriever to miss relevant evidence or return noisy context that confuses generation.

In production, chunking is not just a preprocessing step. It is a retrieval-quality lever that directly influences:

- Recall (can the system find relevant evidence at all?).
- Precision (are returned chunks focused on the user intent?).
- Grounding quality (does the model receive enough context to answer correctly?).
- Cost (how many tokens are sent to rerankers and generators?).

### Why Chunking Is a First-Class Design Decision

Chunk boundaries define what the embedding model "sees" as one semantic unit. If a boundary splits a key claim from its qualifier, embeddings become ambiguous. If a boundary merges unrelated topics, vector similarity can retrieve irrelevant text due to mixed semantics.

This means chunking quality determines whether retrieval candidates are:

- Self-contained enough to support answer generation.
- Focused enough to rank high for the right query.
- Distinct enough to avoid near-duplicate top-k results.

### 1) Fixed-Size Chunking

Fixed-size chunking splits text into uniform windows, usually by token count (for example, 256, 512, or 1024 tokens), sometimes by characters as a simpler approximation.

How it works:

- Choose a chunk length.
- Split text sequentially at that interval.
- Optionally add overlap between consecutive chunks.

Strengths:

- Easy to implement and scale.
- Predictable index characteristics (chunk count and average length).
- Works well for homogeneous prose where topic flow is smooth.

Weaknesses:

- Ignores document structure and sentence boundaries by default.
- Can split definitions, formulas, or code explanations in unnatural places.
- Can merge unrelated content if a section transition occurs inside a fixed window.

Where it fits best:

- Baseline systems and early prototypes.
- Corpora with consistent writing style and low structural complexity.
- Pipelines where operational simplicity is prioritized first.

### 2) Semantic Chunking

Semantic chunking segments text based on meaning shifts rather than fixed length. Typical implementations detect topic transitions using sentence embeddings, heading signals, discourse markers, or cohesion scores.

How it works conceptually:

- Break text into small units (sentences or short paragraphs).
- Compute semantic similarity between adjacent units.
- Start a new chunk when similarity drops below a threshold or a topic break is detected.

Strengths:

- Produces meaning-coherent chunks that align better with user intent.
- Improves retrieval precision when documents contain multi-topic sections.
- Reduces mixed-topic chunks that hurt ranking quality.

Weaknesses:

- More complex to tune than fixed windows.
- Sensitive to threshold settings (too strict creates tiny fragments; too loose merges topics).
- More preprocessing compute and engineering overhead.

Where it fits best:

- Knowledge bases with long narrative documents.
- Policy, legal, or research content where topic boundaries matter.
- Systems where retrieval precision is a primary KPI.

### 3) Recursive Chunking

Recursive chunking applies hierarchical splitting rules from coarse structure to fine granularity. It attempts high-level boundaries first, then progressively smaller separators until chunks satisfy size constraints.

Typical separator order:

- Document sections or headings.
- Paragraph breaks.
- Sentence boundaries.
- Token-length fallback splits.

Strengths:

- Preserves natural structure before forcing token constraints.
- Avoids abrupt cuts when larger semantic boundaries are available.
- Produces robust chunks across mixed-format documents.

Weaknesses:

- Requires thoughtful separator priority design.
- Can behave inconsistently on messy source text with weak formatting.
- Needs guardrails to prevent very short tail chunks.

Where it fits best:

- General-purpose RAG systems handling Markdown, HTML, docs, and notes.
- Corpora with inconsistent formatting but detectable structure.
- Teams seeking a strong default strategy before advanced semantic tuning.

### 4) Sliding Window Chunking

Sliding window chunking generates chunks with a moving window across text. Unlike simple fixed splitting, each next chunk starts before the previous one fully ends, creating intentional continuity.

How it works:

- Define window size (for example, 400 tokens).
- Define stride (for example, 200 tokens).
- Move the window by stride until the document ends.

If stride is smaller than window size, chunks overlap automatically.

Strengths:

- Preserves cross-boundary context for queries that reference transitions.
- Increases chance that a full fact span appears in at least one chunk.
- Works well for procedural or timeline-heavy text.

Weaknesses:

- Increases index size and retrieval redundancy.
- Can return near-duplicate chunks in top-k results.
- Raises retrieval and reranking compute cost.

Where it fits best:

- Content where important information often spans adjacent paragraphs.
- QA tasks that require contextual continuity.
- Systems that can afford additional index/storage overhead.

### 5) Overlap Strategies

Overlap determines how much content is shared between consecutive chunks. It is often the difference between missing and capturing cross-boundary facts.

Why overlap helps:

- Preserves references split at boundaries (for example, "this policy" followed by details).
- Keeps definitions and examples together when boundaries are imperfect.
- Improves robustness against brittle query phrasing.

Common overlap patterns:

- Fixed overlap count (for example, 10-20% of chunk size).
- Structure-aware overlap (repeat last sentence/paragraph when crossing sections).
- Adaptive overlap (higher overlap for dense technical text, lower for short atomic content).

Risks of excessive overlap:

- Duplicate-heavy retrieval results.
- Larger vector index and higher storage cost.
- Reduced diversity in retrieved evidence.

Practical guideline:

- Start with moderate overlap (for example, 15-20% by tokens).
- Evaluate duplicate ratio in top-k retrieval.
- Reduce overlap if redundancy dominates; increase only when boundary misses are frequent.

### 6) Chunk Size vs Retrieval Quality Tradeoff

Chunk size is a direct tradeoff, not a universal constant.

Small chunks (for example, 100-250 tokens):

- Higher precision for narrow, specific queries.
- Better ranking discrimination between nearby topics.
- Greater risk of losing necessary context for complete answers.

Large chunks (for example, 600-1200+ tokens):

- Better local context and self-contained explanations.
- Higher recall for broad conceptual queries.
- More noise per chunk, which can reduce ranking precision.

How this impacts generation:

- Too small: model sees fragmented evidence and may over-infer missing links.
- Too large: model receives diluted context, increasing distraction risk.

Operational cost impact:

- Smaller chunks increase chunk count, index size, and candidate evaluation volume.
- Larger chunks reduce index count but raise prompt token cost per retrieved chunk.

The right size depends on:

- Query shape (fact lookup vs synthesis).
- Content type (API references vs long policy docs).
- Retriever/reranker behavior.
- Generation prompt budget and latency targets.

### 7) Context-Aware Chunking (Sections, Paragraphs)

Context-aware chunking uses document structure signals to form chunks that align with how humans read meaning: by section, subsection, paragraph, list, table, and code block boundaries.

Core principle:

- Preserve semantic units first, then enforce token constraints second.

What to preserve explicitly:

- Heading hierarchy (`h1-h6` or markdown headings).
- Paragraph integrity for argument continuity.
- List blocks so bullets remain grouped.
- Table boundaries and captions.
- Code blocks with nearby explanatory text.

Benefits:

- Better embedding coherence due to cleaner semantic boundaries.
- Better retrieval interpretability (chunks map to recognizable sections).
- Easier citation and provenance because chunk lineage is clear.

Key implementation details:

- Carry structural metadata per chunk (`section_title`, `subsection`, `position`, `source_id`).
- Include parent heading context in chunk text or metadata.
- Avoid merging content across unrelated section boundaries just to hit size targets.

### Recommended Tuning Workflow

Treat chunking as an experimentally tuned subsystem, not a one-time static decision.

Step-by-step approach:

1. Establish a baseline (for example, recursive chunking with moderate overlap).
2. Build an evaluation query set that reflects real user intents.
3. Measure retrieval metrics (precision@k, recall@k, MRR, nDCG) plus answer-level grounding quality.
4. Sweep chunk sizes and overlap values in controlled experiments.
5. Compare error types (boundary misses, topic dilution, duplicate-heavy top-k, missing qualifiers).
6. Lock a default profile per content type, not one global setting for all data.

### Practical Defaults (Then Optimize)

If you need a strong initial setup before deep tuning:

- Start with recursive, structure-first splitting.
- Use 300-700 token chunk targets depending on domain density.
- Apply 15-20% overlap as a starting point.
- Preserve section metadata and source lineage for every chunk.
- Add reranking to improve final candidate ordering when top-k is noisy.

Then iterate using real query logs and failure analysis. The best chunking strategy is the one that improves grounded answer quality on your actual workload, not on synthetic examples.
