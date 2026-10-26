# System: Marketing Automation with AI & Agents

These three ADRs document key architectural decisions for the system, including the available options, trade-offs, and the selected approach.

## ADR-001: LLM Deployment — API vs. Self-Hosted LLM

**Context:** The system uses AI agents to generate marketing content, summarize campaign performance, and personalize customer communications. We need to choose how to deploy and access LLMs.

**Options considered:**

- **Option A — External LLM API:** Use a managed provider such as OpenAI or Anthropic.
- **Option B — Self-hosted LLM:** Run an open-weight model on our own infrastructure.

| Trade-off | External API | Self-hosted |
| --- | --- | --- |
| Initial cost | Low | High |
| Scalability | Easier | Requires capacity planning |
| Maintenance | Provider-managed | Team-managed |
| Data control | Depends on provider and contract | Greater infrastructure control |
| Time to market | Faster | Slower |

**Decision:** Use an external LLM API through a centralized gateway, with authentication, usage monitoring, rate limits, and data-protection controls.

**Rationale:** This supports faster delivery and easier scaling without requiring dedicated GPU infrastructure. Self-hosting can be reconsidered if data residency, customization, or usage economics justify it.

## ADR-002: Knowledge Retrieval — Vector Database Choice

**Accepted**

**Context:** AI agents need to retrieve relevant information from marketing documents, brand guidelines, campaign history, and product descriptions. We need a solution for semantic search to support Retrieval-Augmented Generation (RAG).

**Options considered:**

- **Option A — PostgreSQL with pgvector:** Store embeddings alongside relational data.
- **Option B — Dedicated vector database:** Use a specialized solution such as Pinecone or Qdrant.
- **Option C — Keyword search only:** Retrieve documents using exact words and traditional text matching.

| Trade-off | PostgreSQL + pgvector | Dedicated vector DB | Keyword search |
| --- | --- | --- | --- |
| Operational complexity | Low if PostgreSQL already exists | Additional infrastructure | Low |
| Semantic search | Supported | Supported | Limited |
| Relational integration | Excellent | Requires integration | Depends on implementation |
| Scaling specialized vector workloads | Depends on workload and tuning | Often well suited | Not designed for vector search |

**Decision:** Use PostgreSQL with pgvector for the initial implementation, with tenant-aware filtering to ensure each customer can access only their own data.

**Rationale:** It reduces infrastructure complexity and integrates well with existing relational data. Reassess a dedicated vector database if retrieval latency, dataset size, or workload demands exceed acceptable limits.

## ADR-003: Campaign Execution — Real-Time vs. Batch Processing

**Accepted**

**Context:** The system must support immediate customer interactions, such as responding to incoming messages, as well as large-scale tasks, such as generating campaign content, analyzing engagement, and preparing scheduled reports.

**Options considered:**

- **Option A — Real-time processing:** Process each request immediately as it arrives.
- **Option B — Batch processing:** Collect tasks and process them in scheduled groups.
- **Option C — Hybrid processing:** Use real-time processing for urgent interactions and batch processing for non-urgent workloads.

| Trade-off | Real-time | Batch | Hybrid |
| --- | --- | --- | --- |
| Response latency | Low | Higher | Depends on workload |
| Large-volume efficiency | Can be expensive | Efficient for bulk tasks | Efficient with suitable routing |
| Resource utilization | Demand-driven | Can optimize bulk workloads | More flexible |
| Implementation complexity | Moderate | Moderate | Higher |
| Best use case | Live interactions | Reports and bulk generation | Mixed marketing workloads |

**Decision:** Adopt a hybrid architecture. Use real-time processing for incoming customer messages and time-sensitive agent actions. Use asynchronous queues and batch workers for bulk content generation, analytics, scheduled campaigns, and reporting.

**Rationale:** This balances responsiveness and processing efficiency. Queue monitoring, retry policies, idempotency, and task prioritization will help ensure reliability.

## Summary of decisions

| ADR | Decision | Primary benefit |
| --- | --- | --- |
| ADR-001 | External LLM API via centralized gateway | Faster delivery and easier scaling |
| ADR-002 | PostgreSQL with pgvector | Simpler infrastructure and integrated data |
| ADR-003 | Hybrid real-time and batch processing | Balance responsiveness and efficiency |

**Governance note:** These are proposed decisions for the architecture exercise, marked as accepted to demonstrate a completed governance pack. In a real project, each ADR should be reviewed by the appropriate stakeholders, and its status updated to reflect the actual approval.

# Architecture Review Board

## 1. Security

Protect the platform, customer data, APIs, and AI agents.

- Authentication and role-based access control (RBAC) are enforced.
- Tenant isolation prevents cross-customer data access.
- API keys and credentials are stored securely and rotated.
- Data is encrypted in transit and at rest.
- Prompt injection and malicious document inputs are addressed.
- Agent tools use least-privilege permissions and validate inputs.
- Security logs, alerts, vulnerability scanning, and incident response are in place.

## 2. Privacy & PDPL

Ensure personal data is collected and processed lawfully.

- The purpose and lawful basis for each personal-data processing activity are documented.
- Only necessary personal data is collected and processed.
- Privacy notices and consent mechanisms are provided where required.
- Data-subject rights request procedures are defined.
- Retention periods and secure deletion procedures are documented.
- Cross-border transfers and LLM-provider data handling have been assessed.
- Processors, vendors, and data-processing agreements are reviewed.
- A privacy impact assessment is completed where required.
- Personal-data breach detection, escalation, and notification procedures are defined.

## 3. AI Evaluations (Evals)

Verify that AI outputs are accurate, safe, and fit for purpose.

- Acceptance criteria and measurable quality thresholds are defined for each AI use case.
- A representative evaluation dataset covers normal, edge, and adversarial cases.
- Generated content is tested for factual accuracy, relevance, and brand alignment.
- RAG retrieval quality and groundedness are evaluated where applicable.
- Prompt injection, harmful outputs, and privacy leakage are tested.
- Regression evaluations run when models, prompts, tools, or datasets change.
- Production quality, failure rates, and drift are monitored.
- A rollback or fallback strategy exists when evaluation or production quality degrades.

## 4. Cost & Performance

Keep model usage and infrastructure within agreed budgets.

- Expected request volume, token usage, and infrastructure requirements are estimated.
- Cost per campaign, task, or customer is measurable.
- Budgets, usage quotas, and spending alerts are configured.
- Model selection balances quality, latency, and cost.
- Caching, batching, and asynchronous processing are used where appropriate.
- Rate limits and controls prevent runaway agent loops or repeated tool calls.
- Latency, throughput, and availability targets are defined.
- A lower-cost model or safe fallback is available where appropriate.

## 5. Human Oversight

Ensure humans retain appropriate control over consequential actions.

- Actions requiring human approval are explicitly identified.
- Approval is required before high-impact actions, such as sending sensitive or large-scale campaigns, where appropriate.
- AI-generated marketing content can be reviewed and edited before publication.
- Approvers can inspect relevant content, context, and proposed actions.
- Agent actions and human approvals are recorded in an audit trail.
- Users can stop, cancel, or override agent execution.
- Escalation paths exist for uncertain, harmful, or out-of-policy outputs.
- Ownership and accountability for AI-driven decisions are assigned.
