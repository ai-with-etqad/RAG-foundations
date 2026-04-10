## 🤖 8. Generation Layer (LLM usage)

The generation layer is where retrieved evidence becomes user-facing answers. In production RAG systems, this stage is not just "send chunks to an LLM." It is a controlled synthesis pipeline with strict rules for grounding, citation behavior, uncertainty handling, and decoding choices.

If retrieval determines what evidence is available, generation determines whether that evidence is used faithfully. Most reliability failures in RAG (hallucinated details, missing citations, overconfident tone, weak refusal behavior) happen here.

The goal of a mature generation layer is simple:

1. Use retrieved context as the primary source of truth.
2. Produce clear, useful answers with explicit provenance.
3. Refuse or qualify responses when evidence is weak.
4. Balance factuality, style quality, latency, and cost.

### Prompt Engineering for RAG

Prompt design in RAG is not about generic creativity templates. It is about behavioral control: forcing the model to prioritize evidence, separate known facts from assumptions, and produce outputs in a predictable structure.

A strong RAG generation prompt usually includes:

- **Role and objective:** define the assistant as an evidence-grounded system, not an unconstrained conversational model.
- **Evidence boundaries:** explicitly state that answers must rely on provided context.
- **Fallback behavior:** define what to do when context is insufficient (ask clarification, say unknown, or provide partial answer with caveats).
- **Output contract:** specify format (bullet points, JSON, with citations, confidence tag, etc.).
- **Style constraints:** concise vs detailed, technical depth, audience adaptation.

Why prompt engineering matters so much in RAG:

- Retrieved context can be noisy or partially relevant; the prompt determines evidence prioritization.
- LLMs have a prior to be helpful and complete responses, which can trigger invention when evidence is missing.
- Consistent output formats are required for downstream consumers (UI rendering, evaluators, post-processors).

High-leverage prompt blocks:

1. **System grounding policy:** "Use only retrieved context for factual claims."
2. **Citation instruction:** "Attach source references to each factual statement."
3. **Uncertainty policy:** "If evidence is missing, explicitly say what is unknown."
4. **Conflict policy:** "If sources disagree, surface disagreement and do not collapse into false certainty."

Practical prompt design pattern:

1. Start with strict grounding and short output.
2. Evaluate failure traces (hallucinations, missing citations, wrong confidence).
3. Add targeted instructions for recurring errors.
4. Keep prompt concise enough to avoid token waste and instruction dilution.

### Context Injection Strategies

Context injection is how retrieved chunks are transformed into model input. Quality depends not only on which chunks are selected, but also on ordering, grouping, annotation, and budget control.

Common strategies:

- **Flat concatenation:** append top chunks in rank order. Simple, but often suboptimal when evidence is redundant or mixed quality.
- **Ranked sections:** provide chunks with explicit rank labels and source metadata.
- **Thematic grouping:** cluster by subtopic and inject in grouped blocks for multi-facet questions.
- **Structured evidence block:** encode each chunk as fields (`source`, `title`, `timestamp`, `content`, `score`) before answer generation.
- **Hierarchical injection:** summary-first context followed by high-priority raw passages.

Important context injection decisions:

- **Ordering policy:** highest precision first vs chronology-first vs source-priority.
- **Redundancy trimming:** deduplicate near-identical chunks before prompt assembly.
- **Window allocation:** reserve tokens for instructions, evidence, and answer (not evidence alone).
- **Metadata inclusion:** include source IDs and timestamps to support citation and recency-aware answers.

Why this changes answer quality:

- LLM attention is not uniform across long prompts; earlier and denser evidence often dominates.
- Poor chunk order can cause the model to anchor on weak passages.
- Missing metadata makes citations brittle and prevents conflict-aware reasoning.

A practical production pattern:

1. Keep only high-quality reranked chunks.
2. Remove duplicates and very low-information text.
3. Inject each chunk with stable source identifiers.
4. Add a short "evidence summary" before raw passages for long contexts.

### Grounded Generation (Forcing Model to Stick to Context)

Grounded generation means the model is behaviorally constrained to produce claims that are entailed, or at least strongly supported, by supplied evidence.

Core control mechanisms:

- **Instruction-level grounding:** explicit rules forbidding unsupported claims.
- **Schema-level grounding:** require answer fields tied to evidence IDs.
- **Decode-time grounding:** lower temperature and constrained decoding patterns.
- **Post-generation verification:** check each claim against retrieved context before finalizing response.

Grounding tactics that work in practice:

- **Claim-evidence pairing:** require each key claim to reference one or more chunk IDs.
- **Quote-first synthesis:** extract evidence snippets first, then synthesize.
- **Two-pass generation:** pass 1 generates evidence map; pass 2 writes final answer using only map entries.
- **Refusal policy:** if no supporting evidence exists, output "insufficient evidence" instead of extrapolating.

Failure modes even with grounding prompts:

- Model blends prior knowledge with retrieved evidence.
- Model overgeneralizes from one partial sentence.
- Model fills missing numeric values using plausible defaults.

Mitigations:

1. Tighten instruction language around unsupported inference.
2. Use lower-variance decoding for factual tasks.
3. Add claim-level validators (string overlap, entailment models, or verifier LLM).
4. Penalize unsupported claims in offline eval and online feedback loops.

### Citation Generation

Citations convert RAG from "sounds right" to "traceably right." Strong citation behavior means each factual statement can be traced to source evidence shown in retrieval.

Citation design choices:

- **Granularity:** sentence-level vs paragraph-level citations.
- **Format:** inline markers (`[S1]`, `[S2]`) vs footnotes vs structured JSON references.
- **Strictness:** citation required for all factual claims vs only non-trivial claims.
- **Source fidelity:** references must map to injected chunks, not external model memory.

Reliable citation pipeline pattern:

1. Assign immutable source IDs during context assembly.
2. Instruct the model to cite by those IDs only.
3. Post-validate every citation points to an existing injected source.
4. Optionally verify semantic support between claim and cited text.

Common citation issues:

- **Phantom citations:** model invents source IDs that were never injected.
- **Citation drift:** correct claim paired with wrong source.
- **Citation dumping:** same citation repeated for unrelated statements.
- **Low-granularity mismatch:** one citation cannot support all claims in long paragraphs.

Best practices:

- Keep source IDs short and deterministic.
- Encourage one-to-many mapping only when evidence genuinely spans multiple sources.
- Force explicit "no citation available" for unsupported claims rather than silent omission.
- Evaluate citation precision and recall, not just answer readability.

### Handling Hallucinations

Hallucinations in RAG are usually "grounding failures," not only model defects. They happen when retrieval misses evidence, context is noisy, or generation constraints are weak.

Major hallucination categories:

- **Unsupported factual addition:** claim not present in any context chunk.
- **Wrong synthesis:** individually true fragments combined into false conclusion.
- **Temporal hallucination:** outdated facts presented as current.
- **Numeric hallucination:** fabricated quantities, percentages, dates, or ranges.

System-level anti-hallucination controls:

- **Retrieval confidence gates:** low-confidence retrieval triggers fallback instead of free-form answering.
- **Answerability classification:** detect whether context can answer the question before generation.
- **Conservative prompting:** prefer refusal over speculative completion.
- **Verification loop:** run claim checks before returning the final answer.
- **Human escalation path:** for high-stakes queries, route ambiguous outputs for review.

Product behavior that improves trust:

- Explicitly label uncertainty.
- Separate "what sources state" from "possible interpretation."
- Ask clarifying questions when intent or evidence is ambiguous.
- Avoid confident tone when evidence coverage is partial.

Operational metric set:

- Hallucination rate on adversarial test queries.
- Unsupported-claim count per answer.
- Refusal appropriateness (not only refusal frequency).
- User-reported trust and correction rates.

### Temperature / Decoding Tuning

Decoding parameters strongly affect RAG reliability. For factual QA, low-variance generation usually outperforms creative settings because it reduces speculative continuation.

Primary knobs:

- **Temperature:** lower values reduce randomness; better for grounded factual responses.
- **Top-p (nucleus sampling):** limits token selection to probable mass; lower values improve determinism.
- **Top-k (when available):** restricts candidate token set per step.
- **Max tokens:** caps response length and can reduce rambling unsupported claims.
- **Stop sequences:** enforce output boundaries and formatting contracts.

Typical tuning guidance for RAG:

- Start with low temperature (`0.0` to `0.3`) for evidence-heavy QA.
- Use moderate `top_p` constraints to avoid high-entropy completions.
- Increase creativity settings only for non-factual tasks (tone rewrite, brainstorming).
- Tune per route: strict factual assistant can use different decoding than conversational explainer.

Tradeoff framing:

- Lower randomness improves factual stability and reproducibility.
- Higher randomness can improve fluency variation but increases hallucination risk.
- Very strict decoding can become terse or brittle; use prompt quality and better context before raising temperature.

Production tuning loop:

1. Define evaluation slices (fact QA, multi-hop, ambiguous queries, no-answer queries).
2. Sweep decoding configs and compare groundedness + usefulness.
3. Lock defaults per task type, not one global setting for all routes.
4. Monitor drift when model versions change.

### Models

Model choice in the generation layer is a multi-objective decision: quality, factual obedience, latency, cost, context window, enterprise controls, and ecosystem fit.

There is no single "best model." The right choice depends on workload profile:

- High-stakes factual QA may prioritize groundedness and citation behavior.
- Interactive chat may prioritize latency and conversational quality.
- Cost-sensitive scale workloads may prioritize tokens-per-dollar with strong guardrails.

#### GPT Models

GPT-family models are commonly used in RAG for strong instruction following, tool use patterns, and broad ecosystem support.

Where they typically perform well:

- Structured output with explicit schemas.
- Multi-step synthesis across several context chunks.
- Strong developer tooling and mature API ergonomics.

Operational strengths:

- Robust function/tool-calling patterns in complex agents.
- Good prompt adherence when constraints are clear and concise.
- Broad compatibility with observability, eval, and guardrail tooling.

Watchouts:

- Cost can rise quickly in high-volume long-context workloads.
- Prompt bloat can dilute grounding behavior.
- Citation quality still requires explicit prompt + validation, not assumption.

Recommended usage pattern:

1. Use strict grounded system prompt and source-ID citation format.
2. Keep context curated and reranked before generation.
3. Add verifier or citation validator for high-stakes domains.

#### Claude

Claude-family models are widely used for long-context reasoning, nuanced writing quality, and strong behavior in instruction-heavy enterprise workflows.

Where Claude often stands out:

- Long document reasoning and synthesis over large context windows.
- Natural, clear prose with strong coherence.
- Careful handling of nuanced or policy-heavy prompts.

Operational strengths:

- Good performance on context-rich enterprise knowledge tasks.
- Useful for workflows needing both analysis and polished explanations.
- Often strong at preserving tone constraints with explicit instructions.

Watchouts:

- Long context can tempt over-inclusion; context curation is still mandatory.
- Latency and cost profiles vary by model tier and token footprint.
- Hallucination resistance still depends on grounding instructions and verifier design.

Recommended usage pattern:

1. Use structured evidence blocks with stable source IDs.
2. Require sentence-level citations for factual statements.
3. Apply claim verification on critical outputs.

#### Gemma

Gemma-family models are important in open and local-first stacks where teams need deployment control, customization, and potentially lower serving cost.

Where Gemma is commonly chosen:

- Self-hosted or private deployments.
- Cost-aware workloads with controlled latency environments.
- Domain adaptation pipelines where fine-tuning or adapter strategies are needed.

Operational strengths:

- Flexibility in infrastructure ownership and privacy controls.
- Strong option for teams optimizing custom inference stacks.
- Can be paired with specialized rerankers and validators for robust RAG.

Watchouts:

- Quality varies by model size and serving setup.
- Requires stronger prompt discipline and guardrails for stable grounding.
- Engineering overhead (hosting, scaling, observability) is on your team.

Recommended usage pattern:

1. Use conservative decoding for factual routes.
2. Pair with strict retrieval filtering and reranking.
3. Add post-generation verification to offset smaller-model variance.

#### Qwen

Qwen-family models are frequently used in open-model ecosystems for strong multilingual capability and competitive quality-cost tradeoffs.

Where Qwen often performs well:

- Multilingual RAG workloads and cross-lingual retrieval/generation.
- Cost-efficient deployments with good general reasoning quality.
- Flexible deployment scenarios across cloud and self-hosted setups.

Operational strengths:

- Broad open-model ecosystem momentum and tooling.
- Strong practical performance for many applied assistant tasks.
- Useful choice when balancing capability, openness, and spend.

Watchouts:

- Behavior consistency can differ across Qwen variants and quantizations.
- Requires careful eval for domain-specific groundedness.
- Citation reliability still needs explicit formatting rules and validation.

Recommended usage pattern:

1. Evaluate per-language groundedness, not only aggregate quality.
2. Tune decoding separately for factual vs conversational routes.
3. Add citation and claim-validation checks in production.

### Recommended Production Blueprint for Generation Layer

A stable generation layer in production typically follows this flow:

1. Accept reranked, deduplicated, metadata-rich context.
2. Build prompt with strict grounding policy and citation contract.
3. Generate with low-variance decoding defaults for factual tasks.
4. Validate citations and unsupported claims.
5. Return final answer, or fallback to uncertainty/refusal when evidence is weak.

This blueprint works because it treats generation as a controlled reliability stage, not only text completion. In high-performing RAG systems, "answer quality" is the result of policy, context discipline, model selection, and continuous evaluation working together.
