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
