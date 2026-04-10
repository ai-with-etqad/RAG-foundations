## 📈 14. Real-world Problems to Practice (Where theory becomes deployment skill)

Learning RAG concepts is necessary, but practical strength comes from solving messy, domain-specific problems where data quality, scale, and user intent are imperfect.

This chapter gives you high-value practice projects that mirror real production demand:

1. RAG for PDFs (research papers).
2. RAG for internal company docs.
3. RAG for customer support chatbot.
4. RAG for codebase assistant.

Across all four, you will also train for three universal failure patterns:

- Noisy data.
- Large datasets.
- Ambiguous queries.

The goal is not only to get "an answer" but to build systems that stay grounded, auditable, and useful under real-world constraints.

### Why This Layer Matters

Most RAG projects fail after a good demo because real environments introduce:

- Inconsistent document quality.
- Fast-changing data.
- Conflicting user intent.
- Performance and cost pressure.
- Trust, security, and correctness requirements.

Practicing with realistic problem types builds the engineering muscle to make retrieval quality resilient instead of fragile.

### Practice Methodology (How to use this chapter)

For each project below, train yourself to define:

1. **Success metric:** what "good" means (accuracy, latency, deflection, developer productivity).
2. **Data contract:** expected schema, metadata requirements, and freshness windows.
3. **Retrieval strategy:** chunking, indexing, filtering, and reranking path.
4. **Evaluation set:** representative queries with grounded reference answers.
5. **Failure strategy:** how the system responds when retrieval confidence is low.

If you can articulate and implement these five parts, you are practicing RAG like an engineer, not only like a prompt writer.

### Problem 1: RAG for PDFs (Research Papers)

Research-paper RAG is one of the best training grounds because PDFs combine dense technical language, structure variance, equations, references, and long-context dependencies.

#### Typical Objective

Build a system that can answer questions such as:

- "What is the core contribution of this paper?"
- "How does method A differ from baseline B?"
- "What are the limitations reported in experiments?"
- "Which datasets and metrics were used?"

#### Data and Parsing Realities

PDFs are rarely clean text sources. You should expect:

1. Broken line wraps and hyphenation artifacts.
2. Header/footer duplication on every page.
3. Multi-column extraction errors.
4. Tables and figures that lose structure when converted to plain text.
5. Citation sections that dominate retrieval if not controlled.

A strong pipeline for paper PDFs includes:

- OCR fallback for scanned pages.
- Layout-aware extraction when possible.
- Section segmentation (`abstract`, `method`, `results`, `limitations`).
- Metadata tagging (`title`, `authors`, `year`, `venue`, `section`, `page`).

#### Recommended Retrieval Design

- **Chunking:** section-aware chunks are better than fixed-size only. Keep section boundaries explicit.
- **Metadata filters:** allow query-time filtering by paper/year/section.
- **Hybrid retrieval:** lexical + semantic improves handling for formulas, named methods, and exact metric terms.
- **Reranking:** use cross-encoder reranking for top-k precision on technical questions.
- **Citation-first output:** require page/section citations in final answer format.

#### Evaluation Targets

Your test set should include:

1. Definition questions (concept clarity).
2. Comparison questions (method vs baseline).
3. Evidence questions (where in paper this claim appears).
4. Failure questions (ask for non-existent claims to test hallucination resistance).

#### Common Pitfalls

- Over-chunking by token count and breaking argument flow.
- Retrieval dominated by reference list due to repeated citation tokens.
- Returning claims without evidence page.
- Ignoring equations/tables and then answering confidently.

### Problem 2: RAG for Internal Company Docs

Internal-doc RAG simulates enterprise knowledge systems where quality depends on access control, freshness, and policy-aware retrieval.

#### Typical Objective

Answer internal questions such as:

- "What is our PTO policy for contractors?"
- "How do we onboard a new production service?"
- "What changed in the security incident process last quarter?"
- "Who approves purchases above threshold X?"

#### Data and Governance Realities

Internal corpora usually include:

- Wikis.
- Notion/Confluence pages.
- Docs and slide exports.
- Standard operating procedures.
- Policy PDFs.
- Chat transcripts and meeting notes.

Challenges:

1. Contradictory documents (old policy vs updated policy).
2. Permission-sensitive content by team/role.
3. Rapid updates with stale index risk.
4. Inconsistent authorship and formatting style.

#### Recommended Retrieval Design

- **Document versioning:** mark effective date and last-updated metadata.
- **Access-aware indexing:** propagate role/department/tenant tags to chunk level.
- **Freshness strategy:** prioritize newer docs when policies conflict.
- **Canonical source boosting:** weight official policy docs above informal notes.
- **Conflict-aware responses:** when sources disagree, surface discrepancy explicitly.

#### Evaluation Targets

Include scenario-based tests:

1. Time-sensitive policy queries ("as of this month").
2. Role-specific queries (same question, different allowed answer per role).
3. Contradiction tests (old vs new policy docs).
4. Operational "how-to" questions requiring multi-step retrieval.

#### Common Pitfalls

- Treating all sources as equally trustworthy.
- No freshness weighting, causing outdated answers.
- Missing access filters at retrieval time.
- Citing documents users are not authorized to view.

### Problem 3: RAG for Customer Support Chatbot

Support RAG is excellent practice for relevance, latency, tone control, and safe fallback behavior under high query variability.

#### Typical Objective

Handle customer intents such as:

- "Why was my card charged twice?"
- "How do I reset 2FA if I lost my device?"
- "Where is my order?"
- "How can I cancel and get a refund?"

#### Data and Interaction Realities

Support knowledge sources often include:

- Help-center articles.
- Policy and refund terms.
- Troubleshooting runbooks.
- Product release notes.
- Historical ticket resolutions.

Reality constraints:

1. Users ask incomplete, emotional, or typo-heavy questions.
2. Same intent appears in many phrasings.
3. Incorrect answer can directly harm trust and retention.
4. Escalation to human agents must be smooth and context-preserving.

#### Recommended Retrieval Design

- **Intent + retrieval routing:** classify intent to route to relevant knowledge subset.
- **Short-latency pipeline:** aggressive caching and small top-k before rerank for response speed.
- **Policy certainty checks:** high-risk intents (billing, security, legal) require stricter grounding thresholds.
- **Structured answer templates:** steps, prerequisites, policy conditions, and clear next action.
- **Escalation fallback:** if confidence is low, provide handoff with summarized context.

#### Evaluation Targets

Measure product-level impact, not only text similarity:

1. First-response resolution rate.
2. Human escalation rate (and quality of escalation summary).
3. Policy violation rate in generated answers.
4. User satisfaction proxy (thumbs-up/down or CSAT-linked signals).

#### Common Pitfalls

- Optimizing for fluent language over policy correctness.
- No distinction between informational and transactional intents.
- Missing guardrails for sensitive flows (payments, identity, security).
- No calibrated fallback when confidence is uncertain.

### Problem 4: RAG for Codebase Assistant

Codebase RAG is a high-skill project because correctness depends on repository structure, symbol relationships, and version alignment.

#### Typical Objective

Answer developer questions such as:

- "Where is auth token validation implemented?"
- "How is invoice status computed?"
- "What breaks if I change this interface?"
- "Show me similar examples of this API usage."

#### Data and Code Realities

Code corpora include:

- Source files.
- Tests.
- READMEs and architecture docs.
- API schemas.
- Migration scripts.
- Commit history and PR discussions (optional).

Hard parts:

1. Rapidly changing code invalidates stale embeddings.
2. Semantic meaning depends on imports, symbols, and call graphs.
3. Similar names can exist across services/modules.
4. Wrong guidance can create production bugs.

#### Recommended Retrieval Design

- **Syntax-aware chunking:** chunk by symbol/function/class boundaries, not naive token windows.
- **Rich metadata:** include file path, module, language, symbol name, commit hash/branch.
- **Hybrid search:** combine lexical search for exact identifiers with semantic retrieval for conceptual questions.
- **Dependency-aware reranking:** boost chunks near referenced symbols or call chain.
- **Version pinning:** tie retrieval to commit SHA or branch to prevent mixed-version guidance.

#### Evaluation Targets

Use repository-grounded tasks:

1. Locate implementation questions.
2. Explain behavior with citations to exact files/functions.
3. Refactor-impact questions ("what depends on this?").
4. Regression traps (ensure assistant avoids stale/deleted code references).

#### Common Pitfalls

- Treating code as plain text without symbol boundaries.
- Ignoring branch/commit context in retrieval.
- Returning examples from deprecated modules.
- No confidence signal when evidence is weak.

### Cross-Cutting Challenge 1: Handle Noisy Data

Noise is unavoidable in every real corpus.  
Your system quality depends on whether the pipeline can isolate signal from contamination.

#### What "Noisy" Means in Practice

- Duplicate content across versions.
- Boilerplate/legal text repeated everywhere.
- OCR errors and broken encodings.
- Partial documents and malformed exports.
- Irrelevant but high-frequency terms.

#### Mitigation Strategy

1. **Normalization:** whitespace cleanup, encoding fixes, de-duplication fingerprints.
2. **Boilerplate control:** detect and down-weight repeated low-information segments.
3. **Quality scoring:** assign ingest quality metrics and exclude extremely low-quality chunks.
4. **Metadata enrichment:** preserve source reliability, doc type, and effective dates.
5. **Retrieval filtering:** prefer high-quality, high-authority chunks before generation.

#### Practice Exercise

Create a synthetic noisy dataset by injecting duplicates, OCR-like errors, and outdated versions.  
Benchmark retrieval precision before and after cleanup pipeline changes.

### Cross-Cutting Challenge 2: Handle Large Datasets

Scale pressure appears as both performance and relevance problems:

- Retrieval gets slower.
- Candidate sets get noisier.
- Re-ranking cost increases.
- Index rebuilds become operationally expensive.

#### Scaling Strategy

Use staged retrieval instead of a single heavy query path:

1. Coarse candidate retrieval over partitioned indexes.
2. Metadata filtering to narrow scope early.
3. Lightweight ranking pass.
4. High-quality reranking on small candidate set.
5. Prompt assembly with strict token budget.

#### Index and Ops Tactics

- Shard by tenant/domain/time where appropriate.
- Use hot/warm tiering for frequently accessed vs archival data.
- Adopt incremental indexing and background compaction.
- Track p50/p95/p99 latency by retrieval stage.
- Monitor cost per successful grounded answer.

#### Practice Exercise

Scale corpus size in steps (for example 10k -> 100k -> 1M chunks) and observe:

- Latency curve.
- Retrieval quality drift.
- Cost profile by stage.

Then redesign candidate-generation and rerank budget to recover quality-latency balance.

### Cross-Cutting Challenge 3: Handle Ambiguous Queries

Ambiguity is a core user-behavior reality, not an edge case.

Examples:

- "Explain the policy changes." (which policy?)
- "Fix the issue in auth." (which service? what issue?)
- "How does this work?" (what is "this"?)

#### Ambiguity Handling Strategy

1. **Intent detection:** estimate probable domain(s) and uncertainty score.
2. **Disambiguation prompts:** ask focused clarifying questions when confidence is low.
3. **Multi-hypothesis retrieval:** retrieve candidates across top interpretations before deciding.
4. **Assumption transparency:** explicitly state interpretation when answering.
5. **Safe fallback:** provide possible interpretations rather than hallucinating certainty.

#### Clarification Design Principles

- Ask minimum-friction clarifiers ("Do you mean X or Y?").
- Prefer multiple-choice clarification over open-ended follow-ups.
- Avoid repeated clarification loops; escalate if unresolved.
- Preserve user trust by showing why clarification is required.

#### Practice Exercise

Build an ambiguity benchmark with intentionally underspecified queries.  
Measure:

- Clarification rate.
- Clarification success rate.
- Wrong-assumption answer rate.

### Integrated Practice Roadmap (Recommended order)

Follow this sequence to build real capability progressively:

1. Start with **PDF RAG** for extraction and citation discipline.
2. Move to **internal-doc RAG** for governance, freshness, and access control.
3. Build **support chatbot RAG** for production UX, fallback, and trust constraints.
4. Finish with **codebase assistant RAG** for high-precision technical retrieval.

At each stage, deliberately test noisy data, large dataset behavior, and ambiguous-query handling instead of postponing them.

### Interview-Ready Framing (How to discuss your practice)

When presenting these projects in interviews or architecture reviews, describe each with:

1. Problem context and risk profile.
2. Retrieval and ranking design choices.
3. Evaluation methodology and metrics.
4. Failure handling and fallback behavior.
5. What changed after observing real errors.

This framing shows that you can own RAG systems as products, not only experiments.

### Real-world Practice Checklist

Use this checklist before calling any project "ready":

1. Data contracts and metadata schema are explicit.
2. Retrieval path is measurable and reproducible.
3. Evaluation set includes hard negatives and ambiguity cases.
4. Noisy-data cleanup and quality filters are active.
5. Large-scale performance is tested with staged growth.
6. Ambiguous-query policy includes clarification and safe fallback.
7. Final answers include evidence/citations where domain requires it.

If you can execute this checklist across the four practice problems, you are operating at real-world RAG engineering level rather than tutorial level.
