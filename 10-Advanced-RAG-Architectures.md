## 🔄 10. Advanced RAG Architectures (Once basics are clear, move here)

Once you have a solid baseline RAG pipeline (query understanding, retrieval, reranking, grounded generation, and evaluation), the next performance jump usually comes from architecture, not prompt tweaks.

Advanced RAG architectures exist because many real-world questions are not single-document lookup tasks. They require:

1. Reasoning across multiple sources and steps.
2. Handling structured relationships, not just semantic similarity.
3. Dynamically choosing retrieval or tools at runtime.
4. Preserving context across long conversations and streams.

At this stage, success depends on orchestration quality: how the system plans, retrieves, validates, and synthesizes under uncertainty, cost, and latency constraints.

### Why Advanced Architectures Matter

A basic RAG flow works well for direct factual questions where one or two chunks contain the answer.  
It starts to fail when users ask compositional, temporal, or workflow-heavy questions.

Common signs you need advanced architectures:

- Correct evidence exists, but the model fails to connect it across sources.
- Answers require traversing entities (person -> company -> policy -> timeline).
- Multi-turn chats lose context or repeat retrieval wastefully.
- Users need live updates while retrieval/generation is still running.
- Questions require tools (SQL, APIs, calculators) in addition to text retrieval.

Core shift in mindset:

1. Basic RAG asks: "Can I retrieve relevant chunks?"
2. Advanced RAG asks: "Can I orchestrate reasoning, memory, and tools reliably?"

### Multi-hop RAG (Reason Across Documents)

Multi-hop RAG is designed for questions that need chained evidence from multiple documents, where no single chunk fully answers the query.

#### What It Solves

Single-hop retrieval often returns locally relevant chunks but misses bridge facts required for full reasoning.

Example query type:

- "What policy changed after the acquisition of Company X, and how did that affect region Y operations?"

This requires at least two hops:

1. Find acquisition event details.
2. Find policy changes post-acquisition.
3. Connect policy impact to region Y operations.

#### Typical Pipeline

A production multi-hop pipeline often looks like:

1. **Question decomposition:** split query into intermediate sub-questions.
2. **Hop-1 retrieval:** fetch evidence for first sub-question.
3. **Intermediate inference:** extract bridge entities/facts.
4. **Hop-2+ retrieval:** query again using bridge facts.
5. **Synthesis:** combine evidence into a final grounded answer.
6. **Verification:** ensure each claim has support from one or more hops.

#### Architectural Patterns

- **Sequential hopping:** deterministic chain where each hop depends on previous output.
- **Branch-and-merge:** run multiple second-hop branches in parallel, then merge.
- **Iterative planner loop:** planner decides whether additional hops are needed.
- **Retrieve-read-retrieve loop:** repeated retrieval until confidence threshold is met.

#### Key Design Requirements

- High recall in early hops, otherwise later hops never recover.
- Stable entity extraction between hops (names, dates, IDs).
- Evidence provenance tracking across hops (which claim came from which step).
- Stopping rules to avoid infinite retrieval loops.

#### Failure Modes

- **Bridge entity miss:** first hop misses key connector, collapsing downstream hops.
- **Error propagation:** wrong intermediate inference poisons all future retrieval.
- **Context overflow:** too many hop artifacts exceed generation context budget.
- **Over-hopping:** unnecessary hops increase latency and cost with no quality gain.

#### Production Best Practices

1. Use decomposition prompts or a small planner model for sub-question generation.
2. Keep hop artifacts structured (`sub_question`, `evidence_ids`, `entities`, `confidence`).
3. Deduplicate and rerank evidence at every hop, not only at the end.
4. Add claim-level support checks before final response.
5. Set strict max-hop and max-latency budgets.

#### When To Use

Use multi-hop RAG when user queries frequently require combining facts across documents, timelines, or domains.  
If most queries are direct lookups, this may add unnecessary complexity.

### Graph RAG (Knowledge Graphs)

Graph RAG combines retrieval with graph-structured knowledge, enabling reasoning through explicit entities and relationships, not only vector similarity.

#### What It Solves

Vector retrieval is good at semantic closeness but weak at explicit relational traversal.

Graph RAG helps when questions involve:

- Entity relationships ("Who reports to whom?")
- Dependency paths ("What service depends on this deprecated API?")
- Multi-step lineage ("Where did this metric originate?")
- Compliance or governance links ("Which policy applies to this asset type?")

#### Core Architecture

A practical Graph RAG stack usually contains:

1. **Entity and relation extraction:** convert corpus into nodes and edges.
2. **Graph store:** persist knowledge graph in a graph database or index.
3. **Hybrid retriever:** combine graph traversal + vector/keyword retrieval.
4. **Subgraph builder:** fetch local neighborhood relevant to the query.
5. **Context constructor:** serialize subgraph + supporting text chunks for generation.

#### Retrieval Strategies in Graph RAG

- **Node-first retrieval:** identify primary entities, then expand neighbors.
- **Path-constrained retrieval:** retrieve specific relation paths (A -> B -> C).
- **Community retrieval:** use clusters/subgraphs by topic or domain.
- **Hybrid graph+vector:** vector search for recall, graph traversal for precision and explainability.

#### Why Graph RAG Improves Reasoning

- Makes latent relationships explicit and traversable.
- Improves explainability through paths, not only ranked chunks.
- Supports compositional queries where relation structure is central.
- Helps reduce semantic false positives from pure embedding similarity.

#### Failure Modes

- **Graph staleness:** extracted graph lags behind document updates.
- **Schema drift:** relation types become inconsistent across ingestion pipelines.
- **Noisy extraction:** incorrect edges create plausible but wrong reasoning paths.
- **Serialization loss:** rich graph signals get flattened poorly in prompts.

#### Production Best Practices

1. Define strict ontology/schema for entities and relations early.
2. Version and timestamp graph updates for freshness-aware retrieval.
3. Keep confidence scores for extracted edges and filter low-confidence paths.
4. Merge graph context with raw text evidence so the model can validate claims.
5. Evaluate path correctness, not just final answer fluency.

#### When To Use

Graph RAG is strongest when your domain is relation-heavy: enterprise knowledge maps, biomedical literature, legal citations, dependency graphs, and governance systems.

### Agentic RAG (LLM Decides Retrieval Strategy)

Agentic RAG introduces planning and decision-making where the LLM (or controller model) chooses how to retrieve, when to retrieve, and which tools/routes to call.

#### What It Solves

Static RAG pipelines apply one retrieval strategy to every query.  
Real traffic is diverse: some questions need keyword search, some need graph traversal, some need SQL/API tools, and some need no retrieval at all.

Agentic RAG adapts strategy per query.

#### Typical Agentic Loop

A common controller loop:

1. **Route classification:** decide query type and difficulty.
2. **Plan generation:** choose steps (retrieve, tool-call, re-query, clarify).
3. **Action execution:** run selected retrievers/tools.
4. **Observation:** inspect retrieved evidence and tool outputs.
5. **Replan or finalize:** either continue loop or generate answer.

#### Decision Dimensions

The agent typically decides:

- Which retriever to use (dense, sparse, hybrid, graph).
- How many retrieval iterations are needed.
- Whether to call external tools before/after retrieval.
- Whether to ask user clarifying questions.
- Whether to refuse due to insufficient evidence.

#### Architectural Components

- **Policy layer:** prompt- or model-based decision rules.
- **Action registry:** allowed retrieval and tool actions.
- **State memory:** running context of plans, evidence, and observations.
- **Budget controller:** max steps, token budget, timeout constraints.
- **Safety/guardrails:** prevent unsafe tool calls or unsupported outputs.

#### Failure Modes

- **Plan thrashing:** too many replans with minimal quality gain.
- **Tool overuse:** agent calls expensive tools unnecessarily.
- **Unstable routing:** similar queries take inconsistent paths.
- **Opaque behavior:** hard to debug why a particular action was chosen.

#### Production Best Practices

1. Start with constrained agent policies before fully open-ended actions.
2. Log every decision step (`reason`, `action`, `inputs`, `outputs`, `cost`, `latency`).
3. Add deterministic fallbacks for common query classes.
4. Enforce hard limits on steps, time, and tool invocations.
5. Evaluate route quality, not only final answer quality.

#### When To Use

Use agentic RAG when query diversity is high and fixed pipelines leave obvious quality gaps.  
Avoid it for narrow workflows where deterministic orchestration is simpler and more reliable.

### Conversational RAG (Chat History Aware)

Conversational RAG adapts retrieval and generation using conversation memory so answers remain coherent across turns, references, and evolving user intent.

#### What It Solves

Standard RAG treats each message in isolation.  
In real chats, users ask follow-ups like "What about the second option?" or "Compare that with last year's policy."

Without conversation-aware retrieval:

- Core entities are lost between turns.
- Pronouns and ellipsis become ambiguous.
- The system re-retrieves redundant context each turn.
- Long chats degrade quality due to memory drift.

#### Core Pipeline

A robust conversational pipeline often includes:

1. **Turn understanding:** detect intent and references to prior context.
2. **Query rewriting:** transform follow-up into standalone retrievable query.
3. **Memory selection:** choose relevant history (not full transcript dump).
4. **History-aware retrieval:** combine rewritten query + selected memory cues.
5. **Grounded response generation:** answer current turn with citations and continuity.

#### Memory Layers

- **Short-term memory:** recent turns for immediate continuity.
- **Long-term memory:** persistent user/profile/task preferences.
- **Semantic memory index:** searchable summaries of prior conversation states.
- **Session state memory:** active entities, decisions, unresolved questions.

#### Design Trade-offs

- More history improves continuity but increases noise and token cost.
- Aggressive summarization reduces cost but may lose critical details.
- Personalization improves UX but adds privacy and governance requirements.

#### Failure Modes

- **History pollution:** irrelevant old turns bias current answers.
- **Reference resolution errors:** "it/that" linked to wrong entity.
- **Memory hallucination:** model treats prior speculative content as facts.
- **Cross-session leakage:** wrong user/session context appears in answers.

#### Production Best Practices

1. Use query rewriting before retrieval for follow-up turns.
2. Maintain explicit entity/state trackers, not only raw transcript memory.
3. Separate factual memory from conversational style/preference memory.
4. Apply strict user/session isolation and privacy controls.
5. Periodically summarize long sessions and revalidate key facts.

#### When To Use

Conversational RAG is essential for assistants with ongoing multi-turn workflows: support copilots, research assistants, tutoring systems, and domain advisors.

### Streaming RAG

Streaming RAG provides incremental retrieval and generation so users see useful output quickly instead of waiting for full pipeline completion.

#### What It Solves

In slower pipelines, users experience dead time while retrieval, reranking, and synthesis finish.  
Streaming improves perceived responsiveness and supports progressive disclosure of results.

#### Streaming Architecture Pattern

A common pattern:

1. **Early retrieval pass:** fetch high-confidence initial evidence quickly.
2. **Initial answer draft stream:** start response with clearly labeled partial grounding.
3. **Background enrichment:** continue retrieval/reranking for deeper evidence.
4. **Answer refinement stream:** append updates, corrections, and additional citations.
5. **Finalization:** mark stable final answer state.

#### Streaming Modes

- **Token streaming:** stream generated tokens as they are produced.
- **Chunk streaming:** stream validated answer segments.
- **Evidence streaming:** show retrieved sources before final synthesis.
- **Hybrid streaming:** stream answer + source updates in parallel.

#### UX and Reliability Concerns

- Users must know whether output is preliminary or final.
- Later evidence may contradict earlier streamed claims.
- Citation mapping must remain consistent as content updates.
- Interrupt and cancel behavior must be deterministic.

#### Failure Modes

- **Premature certainty:** early streamed claims sound final without full evidence.
- **Revision confusion:** users miss corrections that arrive later.
- **Source mismatch:** citations shift during incremental reranking.
- **State race conditions:** retrieval and generation updates arrive out of order.

#### Production Best Practices

1. Label phases clearly (`draft`, `updating`, `final`).
2. Prefer conservative language in early stream segments.
3. Preserve stable source IDs across all streamed revisions.
4. Buffer and reorder async events before user display when needed.
5. Log stream revisions for auditability and debugging.

#### When To Use

Streaming RAG is ideal for latency-sensitive experiences where fast feedback matters: support chat, research copilots, and long-running retrieval workflows.

### Tool-Augmented RAG

Tool-augmented RAG combines document retrieval with external tools (databases, APIs, calculators, code execution, search endpoints) so the system can answer beyond static corpus knowledge.

#### What It Solves

Text retrieval alone cannot always provide:

- Real-time values (prices, metrics, inventory, weather, status pages).
- Precise computations (financial math, aggregations, simulations).
- Transactional data from structured systems.
- Action-oriented workflows (ticket creation, report generation).

Tool augmentation expands RAG from "retrieve and answer" to "retrieve, compute, validate, and answer."

#### Common Tool Categories

- **Structured query tools:** SQL/warehouse connectors.
- **API tools:** internal service APIs, public endpoints.
- **Computation tools:** calculator, Python runtime, stats engine.
- **Search tools:** web/domain search for freshness.
- **Action tools:** workflow systems, ticketing, notifications.

#### Orchestration Patterns

- **Retrieve-then-tool:** use docs to identify what to query via tools.
- **Tool-then-retrieve:** fetch fresh structured data, then enrich with documentation.
- **Parallel fan-out:** call multiple tools/retrievers concurrently, then fuse.
- **Verifier tool pass:** run external checks before final answer.

#### Trust and Governance Requirements

- Tool permissions must be explicit and least-privilege.
- Inputs to tools should be validated and sanitized.
- Outputs need schema checks before generation.
- High-impact actions require confirmation and audit trails.

#### Failure Modes

- **Tool-result hallucination:** model claims tool output without actual call.
- **Schema mismatch:** model misreads tool response fields.
- **Stale vs fresh conflict:** tool data disagrees with static docs.
- **Unsafe actioning:** unintended side effects from poorly constrained tools.

#### Production Best Practices

1. Require tool-call traceability in final responses when relevant.
2. Use typed schemas and strict parsing for all tool IO.
3. Reconcile conflicts between retrieved docs and live tool outputs explicitly.
4. Separate read-only tools from action tools with stronger confirmation gates.
5. Add circuit breakers, retries, and fallback logic per tool.

#### When To Use

Use tool-augmented RAG when answers require live data, precise computation, or operational actions.  
For purely static knowledge Q&A, standard RAG is often simpler and cheaper.

### Cross-Cutting Architecture Guidelines

No matter which advanced architecture you adopt, production reliability usually depends on these shared disciplines:

1. **Observability:** trace every retrieval hop, route decision, tool call, and citation.
2. **Evaluation by slice:** benchmark simple, multi-hop, conversational, and tool-heavy query classes separately.
3. **Budget control:** enforce latency, token, and tool cost guardrails.
4. **Fallback behavior:** define deterministic fail-safe paths for low-confidence situations.
5. **Data freshness strategy:** align document/graph/tool update frequencies with business requirements.
6. **Security and privacy:** isolate sessions, control tool permissions, and audit sensitive flows.

### Choosing the Right Architecture

A practical selection heuristic:

- Choose **Multi-hop RAG** when evidence must be chained across documents.
- Choose **Graph RAG** when relationship traversal is central to correctness.
- Choose **Agentic RAG** when query diversity requires dynamic routing/planning.
- Choose **Conversational RAG** when multi-turn continuity is a core product behavior.
- Choose **Streaming RAG** when responsiveness and progressive output are critical.
- Choose **Tool-augmented RAG** when static corpus evidence is insufficient alone.

In many mature systems, you do not pick only one.  
You combine them selectively by route, with strict evaluation and guardrails.

### Final Takeaway

Advanced RAG is not about adding complexity for its own sake. It is about matching architecture to question complexity.

Teams that stay with basic RAG for advanced workloads usually compensate with prompt hacks and still hit quality ceilings.  
Teams that introduce the right advanced architecture, with clear orchestration and evaluation discipline, unlock better factuality, better user trust, and better product reliability at scale.
