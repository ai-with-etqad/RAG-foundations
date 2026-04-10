## 🔐 12. Security & Privacy (Critical for production trust)

RAG systems do not only answer questions; they move sensitive data through multiple stages:

1. Ingestion and storage.
2. Indexing and retrieval.
3. Prompt construction and generation.
4. Logging, monitoring, and debugging.

Each stage can leak information if security and privacy are not designed explicitly.  
This is especially important for local-model deployments, where teams often assume "local = automatically safe," while practical risk still depends on access controls, endpoint hardening, and data handling policy.

Security and privacy for RAG is the discipline of protecting:

- **Confidentiality:** only authorized actors can access data.
- **Integrity:** retrieved context and outputs are not tampered with.
- **Availability:** safety controls do not collapse under real traffic.
- **Compliance:** handling aligns with legal and contractual obligations.

### Why This Layer Matters

RAG increases attack surface compared to plain LLM chat because:

- It touches proprietary document stores.
- It performs retrieval over potentially regulated data.
- It can expose hidden corpus snippets via model outputs.
- It usually logs rich artifacts for observability.
- It often mixes users, roles, and tenants at scale.

Common misconception:

1. "If we use a local model, privacy is solved."
2. "If the model is private, documents are safe."
3. "If the answer looks harmless, no leakage happened."

All three are incomplete.  
Real protection requires end-to-end controls across storage, retrieval, generation, transport, and operations.

### Data Leakage Risks

Data leakage is any unintended exposure of private, restricted, or sensitive information to unauthorized users, systems, or logs.

#### Where Leakage Happens in RAG

High-risk leakage points include:

1. **Ingestion pipelines:** raw documents copied into staging buckets, temp files, or debug queues.
2. **Vector indexes:** chunks embedded and stored without proper tenant/role boundaries.
3. **Retrieval responses:** retriever returns unauthorized chunks before final filtering.
4. **Prompt assembly:** hidden metadata (IDs, emails, internal notes) injected into prompts.
5. **Generation output:** model reproduces sensitive context verbatim.
6. **Application logs:** full prompts/responses logged in plaintext for debugging.
7. **Cache layers:** shared cache keys accidentally return prior user artifacts.
8. **Analytics exports:** BI pipelines receive raw query/context payloads.

#### Leakage Threat Patterns

- **Cross-tenant leakage:** one customer receives another customer's context.
- **Role-bypass leakage:** a user asks broadly and receives admin-only documents.
- **Prompt injection exfiltration:** malicious text asks model to reveal hidden context.
- **Training contamination leakage:** sensitive data appears in future model behavior due to improper fine-tuning dataset hygiene.
- **Operational leakage:** support staff or dashboards expose raw artifacts beyond need-to-know.

#### Practical Controls for Leakage Prevention

Implement controls in layers, not as a single gate:

1. **Pre-ingestion classification:** label documents by sensitivity and policy class before indexing.
2. **Document minimization:** avoid indexing fields not required for retrieval quality.
3. **Chunk sanitation:** strip secrets/tokens from chunks before embedding.
4. **Retrieval-time authorization:** enforce policy before candidate chunks are exposed to prompt builder.
5. **Output guardrails:** detect and redact sensitive spans in generated text before delivery.
6. **Scoped logging:** never store raw full prompts by default; keep masked/hashed artifacts.
7. **Cache segmentation:** key caches by tenant, role, policy version, and model version.

#### Leakage Monitoring and Incident Readiness

Security posture is weak without detection. Track:

- Sensitive-token appearance rate in outputs.
- Unauthorized retrieval attempts by actor and route.
- Policy-denied chunk counts.
- Cross-tenant cache miss/hit anomalies.
- Redaction trigger rates and false negatives.

Run periodic red-team prompts that intentionally attempt exfiltration.  
If leakage is detected, incident workflow should include rapid key rotation, index quarantine, cache invalidation, and scoped user notification based on impact.

### Access Control for Documents

Access control in RAG must be enforced at document level, chunk level, and query-time decision level.

#### Core Principle: Retrieval Must Be Policy-Aware

If access control is applied only after retrieval, leakage risk remains high because sensitive chunks may already enter intermediate systems.

Policy-aware retrieval means:

1. Candidate selection is filtered using user identity + role + tenant + attributes.
2. Reranker operates only on authorized candidates.
3. Prompt builder receives only policy-approved context.

#### Authorization Models

Use one or combine several:

- **RBAC (Role-Based Access Control):** permissions mapped to roles like `employee`, `manager`, `admin`.
- **ABAC (Attribute-Based Access Control):** policies use attributes (department, region, clearance, project).
- **ReBAC (Relationship-Based Access Control):** permissions derived from graph relationships (owner, team member, delegate).

For enterprise RAG, ABAC/ReBAC often handle real-world complexity better than pure RBAC.

#### Policy Propagation Through the Pipeline

Security labels must survive every transformation:

1. Raw document metadata includes policy tags.
2. Chunking stage propagates inherited policy labels.
3. Embedding/index records carry immutable access metadata.
4. Retrieval queries include actor context claims.
5. Downstream caches and logs preserve redaction or policy markers.

If policy tags are dropped at any stage, your enforcement becomes inconsistent and brittle.

#### Multi-Tenant Isolation Requirements

For multi-tenant systems, enforce separation by design:

- Tenant-scoped namespaces or physically separate indexes.
- Tenant-aware encryption keys.
- Strict query filters with server-side validation (never trust client-provided tenant IDs).
- Tenant-scoped cache and session artifacts.
- Admin tooling with explicit tenant context and audited access.

#### Auditability and Forensics

Access controls are only defensible if you can prove what happened:

- Record actor ID, policy decision, document IDs considered, and reason for allow/deny.
- Keep tamper-evident audit logs with retention policy.
- Support replay of authorization decisions for incident analysis.
- Add periodic policy regression tests as access rules evolve.

### On-Device RAG vs Cloud RAG

Both deployment models can be secure, but they fail in different ways.  
Choose based on threat model, compliance needs, operational maturity, and latency/cost trade-offs.

#### On-Device RAG (Local Models + Local Data)

Typical strengths:

- Data stays near endpoint, reducing external transmission exposure.
- Better fit for air-gapped or strict sovereignty environments.
- Lower dependency on external API data policies.

Typical risks:

1. Endpoint compromise (malware, stolen laptop, weak disk protection).
2. Local privilege abuse (other users/processes reading model/index files).
3. Weaker central governance and patch discipline across devices.
4. Difficult incident visibility due to fragmented telemetry.

Hardening requirements:

- Full-disk encryption and secure boot.
- OS-level account isolation and least privilege.
- Encrypted local vector stores and keychain-backed secret storage.
- Signed model and index artifacts to prevent tampering.
- Remote wipe, device compliance checks, and patch enforcement.

#### Cloud RAG (Managed or Self-Hosted Cloud)

Typical strengths:

- Centralized governance, patching, and policy enforcement.
- Better observability, auditing, and incident response.
- Easier key rotation and secrets lifecycle control.
- Scalable IAM integration with enterprise identity systems.

Typical risks:

1. Misconfigured cloud permissions exposing storage/index endpoints.
2. Third-party processor risk and jurisdictional concerns.
3. Public network exposure if transport/auth are weak.
4. Shared infrastructure complexity and dependency risk.

Hardening requirements:

- Private networking and strict ingress/egress controls.
- Strong IAM boundaries with least privilege service roles.
- KMS/HSM-managed encryption keys and rotation policy.
- Region-aware data residency controls.
- Vendor due diligence and contractual data processing controls.

#### Hybrid Pattern (Common in Practice)

Many teams run:

- Local retrieval over sensitive corpus subsets.
- Cloud generation for non-sensitive or sanitized contexts.
- Policy engine deciding route by data sensitivity and user role.

Hybrid works well only when classification and routing are deterministic and auditable.  
If routing is ambiguous, leakage risk rises quickly.

### PII Handling

PII (Personally Identifiable Information) must be treated as a lifecycle problem, not a one-time redaction step.

#### PII Categories Relevant to RAG

PII may appear in:

- Direct identifiers (name, phone, email, national ID).
- Quasi-identifiers (location, job title, rare combinations).
- Sensitive attributes (health, finance, legal records).
- Free-form notes where PII appears unpredictably.

RAG increases risk because semantic retrieval can surface PII from distant or unrelated documents if indexing is broad and policies are weak.

#### PII Handling Strategy by Stage

1. **Ingestion:** detect and tag PII before indexing.
2. **Storage:** separate high-risk fields where possible.
3. **Indexing:** avoid embedding unnecessary raw identifiers.
4. **Retrieval:** enforce policy-aware filtering for PII sensitivity class.
5. **Prompting:** minimize sensitive fields included in context.
6. **Generation:** apply output redaction and policy checks.
7. **Logging:** mask, tokenize, or avoid storing raw PII in traces.

#### Detection and Redaction Techniques

- Rule-based detectors for known formats (emails, phone numbers, IDs).
- NER/ML-based detectors for contextual PII.
- Domain-specific recognizers (medical, HR, financial identifiers).
- Confidence thresholding with human-review path for high-impact workflows.

Redaction modes:

- **Masking:** partial hide (`a***@domain.com`).
- **Tokenization:** replace with reversible token managed in secure vault.
- **Pseudonymization:** replace identifiers while preserving analytics utility.
- **Deletion:** remove entirely when not required downstream.

#### Data Retention and User Rights

Privacy requirements usually include:

- Data minimization by default.
- Defined retention windows for raw artifacts.
- Right-to-delete workflows that cascade across raw stores, indexes, caches, and backups.
- Access/export workflows for regulated user requests.

If deletion does not propagate to embeddings and snapshots, compliance claims may be invalid.

### Encryption Strategies

Encryption is foundational, but only effective when key management and scope boundaries are engineered correctly.

#### Encryption in Transit

Protect all network paths:

- TLS for client-to-app, app-to-retriever, and app-to-model traffic.
- Mutual TLS for internal service-to-service paths where feasible.
- Certificate rotation and revocation procedures.
- Strict transport policy to prevent downgrade attacks.

Do not assume internal networks are trusted.

#### Encryption at Rest

Apply at each storage layer:

1. Object/document storage.
2. Vector index and metadata store.
3. Relational stores for sessions and policy data.
4. Backups, snapshots, and export bundles.
5. Local endpoint storage for on-device deployments.

Prefer envelope encryption with centralized key management instead of ad-hoc per-service secrets.

#### Key Management and Rotation

Strong encryption fails if key lifecycle is weak.  
Minimum key management controls:

- Keys stored in KMS/HSM, not in source code or config files.
- Role-scoped key access policies.
- Regular rotation schedule and emergency rotation playbook.
- Key versioning for smooth re-encryption migrations.
- Audit logs for every key usage event.

#### Tenant-Aware and Field-Level Encryption

For high-sensitivity systems:

- Use tenant-specific data keys to reduce blast radius.
- Use field-level encryption for especially sensitive columns (for example national IDs, health fields).
- Isolate encryption contexts by environment (`dev`, `staging`, `prod`) and purpose.

This limits impact if one key, store, or environment is compromised.

#### Encryption Trade-offs You Must Plan For

- Extra latency for decrypt/re-encrypt paths.
- Search limitations on encrypted fields.
- Operational complexity in rotation and re-indexing.
- Debugging challenges when plaintext visibility is restricted.

Design observability that remains useful without exposing raw sensitive payloads.

### Security & Privacy Operating Model

Controls become reliable only when embedded into daily operations.

#### Governance Checklist

1. Security owner and privacy owner defined.
2. Data classification policy mapped to system behavior.
3. Threat model reviewed at architecture milestones.
4. Access policy tests run in CI for critical routes.
5. Incident response runbooks rehearsed quarterly.

#### Testing and Validation

Run continuous validation, not one-time audits:

- Prompt-injection and exfiltration tests.
- Cross-tenant isolation tests.
- Policy regression tests for allow/deny boundaries.
- Redaction quality benchmarks (precision/recall).
- Encryption posture verification and key access audit checks.

#### Minimal "Do Not Ship Without" Baseline

- Retrieval-time access control enforced server-side.
- Sensitive logging disabled or masked by default.
- PII detection + output redaction active in production.
- Encryption in transit and at rest with managed keys.
- Audit trail for access decisions and incident triage.

Without these, RAG can provide useful answers but still fail basic trust and compliance expectations.

