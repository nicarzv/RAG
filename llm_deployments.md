# LLM Deployment Strategy — Sweden Data Zone Provisioned

**Status:** Proposal for team review  
**Region:** Sweden Central  
**Deployment type:** Data Zone Provisioned Managed (spillover to Data Zone Standard)  
**Date:** April 2026

---

## 1. Context and Requirements

We need to provision LLMs for our RAG chatbot built on the GPT-RAG Solution Accelerator. The orchestrator uses maf_lite, which calls the LLM directly via the Chat Completions API for streaming responses.

**User base:** ~30 users during the testing phase.

**Document corpus:**
- 4,000 documents ingested in the Azure AI Search index
- Average pages per document: 100
- Median pages per document: 50

**Data residency constraint:** All data must stay within the EU data zone. This means we need models available on both Data Zone Provisioned (for the primary deployment) and Data Zone Standard (for spillover), both in Sweden Central.

**Spillover:** When the provisioned deployment hits 100% utilization, excess traffic is automatically routed to a Data Zone Standard deployment of the same model within the same Azure OpenAI resource. This keeps data within the EU zone while handling traffic bursts. Spillover is configured via the `spilloverDeploymentName` deployment property or per-request via the `x-ms-spillover-deployment` header.

---

## 2. Selection Criteria

Models were evaluated against these criteria:

1. **Data Zone availability** — must be available on both DZ Provisioned and DZ Standard in Sweden Central for safe spillover within the EU
2. **Chat Completions API support** — the orchestrator (maf_lite) uses this API for streaming completions
3. **Throughput efficiency** — input tokens per minute (TPM) per PTU, and the output-to-input token ratio
4. **Latency** — tokens per second (TPS) target for streaming responses to users
5. **Context window** — must accommodate system prompt + retrieved chunks + chat history
6. **Quality for RAG** — instruction following, grounding in retrieved context, structured output support
7. **Cost at minimum scale** — PTU requirements for ~30 test users across realistic workloads

---

## 3. Recommended Models

### 3.1 gpt-4.1 — Recommended (best quality-to-cost ratio)

| Attribute | Value |
|-----------|-------|
| Input TPM per PTU | 3,000 |
| Output token ratio | 1 output = 4 input tokens |
| Context window | 300,000 (provisioned deployments) |
| Max output tokens | 32,768 |
| Latency target | 80 tokens/second (p99) |
| Min PTU (DZ) | 15 |
| DZ Provisioned Sweden | Yes |
| DZ Standard Sweden | Yes |
| Spillover support | Yes |

**Why this model:** gpt-4.1 is the most proven model for RAG workloads. It has excellent instruction following, structured outputs, tool calling, and vision support. The 300K context on provisioned deployments is more than sufficient for any RAG scenario. At 80 TPS latency, users get fast streaming responses. This is the model we would put into production first.

### 3.2 gpt-4.1-mini — Cost efficient (5x throughput per PTU)

| Attribute | Value |
|-----------|-------|
| Input TPM per PTU | 14,900 |
| Output token ratio | 1 output = 4 input tokens |
| Context window | 300,000 (provisioned deployments) |
| Max output tokens | 32,768 |
| Latency target | 90 tokens/second (p99) |
| Min PTU (DZ) | 15 |
| DZ Provisioned Sweden | Yes |
| DZ Standard Sweden | Yes |
| Spillover support | Yes |

**Why this model:** Nearly 5x the throughput per PTU compared to gpt-4.1 (14,900 vs 3,000), with the same feature set and same context window. For RAG specifically, the quality gap between 4.1 and 4.1-mini is smaller than expected — when the answer is grounded in retrieved chunks, the model mostly needs to synthesize and present, not reason from scratch. This is the cost-efficient option: same PTU budget handles far more concurrent users, or the same workload requires far fewer PTUs. If testing shows the answer quality is acceptable for our users, this saves significant cost at scale.

### 3.3 gpt-5.1 — Most capable (latest GA with reasoning)

| Attribute | Value |
|-----------|-------|
| Input TPM per PTU | 4,750 |
| Output token ratio | 1 output = 8 input tokens |
| Context window | 400,000 |
| Max output tokens | 128,000 |
| Latency target | 50 tokens/second (p99) |
| Min PTU (DZ) | 15 |
| DZ Provisioned Sweden | Yes |
| DZ Standard Sweden | Yes |
| Spillover support | Yes |

**Why this model:** The latest GA model with both DZ Provisioned and DZ Standard availability in Sweden. It has configurable reasoning via `reasoning_effort` (defaults to `none`, so it behaves like a standard chat model unless explicitly turned up per request). The 400K context window and 128K max output give headroom for complex multi-turn conversations or very long summaries. The throughput per PTU (4,750) is slightly better than gpt-4.1, but the output ratio is 1:8 instead of 1:4, so output-heavy workloads like summarization are relatively more expensive. Worth testing for queries where gpt-4.1 might miss subtleties or nuance.

---

## 4. Models Considered and Rejected

### gpt-5.4 and gpt-5.2 — No spillover path in Sweden

Both models are available on Data Zone Provisioned in Sweden Central but are **not** available on Data Zone Standard in Sweden Central. This means overflow traffic would have nowhere to go within the EU data zone. Given our data residency constraint, these models cannot be safely deployed with spillover.

### gpt-5 — Superseded by gpt-5.1

Same architecture generation as gpt-5.1 but older. gpt-5.1 has the same availability, better capabilities, and configurable reasoning. No reason to choose gpt-5 over gpt-5.1.

### o3, o4-mini, o1, o3-mini — Reasoning models, overkill for RAG

These are reasoning models designed for complex math, coding, and multi-step logic problems. They are massively overkill for RAG where the model's job is to synthesize retrieved context into a grounded answer. The throughput per PTU is much lower (o1 is only 230 input TPM per PTU vs 3,000 for gpt-4.1) and the cost is significantly higher for the same workload.

### gpt-4o and gpt-4o-mini — Previous generation

Superseded by the 4.1 series in every dimension: lower throughput per PTU (2,500 for gpt-4o vs 3,000 for gpt-4.1), slower latency target (25 TPS vs 80 TPS), and smaller context window (128K vs 300K). No reason to choose them when gpt-4.1 and gpt-4.1-mini are available.

### gpt-4.1-nano — Quality too low for user-facing chatbot

Extremely high throughput (59,400 input TPM per PTU) but noticeably lower quality. Suitable for classification or simple extraction tasks, but not for a user-facing chatbot where answer quality directly impacts user trust and adoption.

### gpt-5.1-codex — Code-optimized, no DZ Standard spillover

Optimized for Codex CLI and VS Code extension. Not designed for RAG chat workloads. Also not available on Data Zone Standard in Sweden, so no safe spillover path.

---

## 5. Workload Estimation

### 5.1 Assumptions

**Users:** 30 test users. Not all users query simultaneously. Peak scenario assumes roughly a third of users active within the same minute.

**Token profile per use case:**

The input token count for RAG chat is driven by the system prompt (~300 tokens), retrieved search chunks from Azure AI Search, chat history from the current conversation turn (~700 tokens), and the user's query (~100 tokens). The search chunks are the dominant factor — typically `top_k` results of 3–5 chunks at ~500–800 tokens each.

**Important caveats on these estimates:**

- The input token estimate for RAG chat (4,000) assumes `top_k` of ~5 and average chunk sizes of 500–800 tokens. If `top_k` is higher or chunks are larger (our `CHUNK_SIZE` and `TOKEN_OVERLAP` settings affect this), the actual input tokens could be closer to 5,000–6,000. The current ingestion pipeline uses `TOKEN_OVERLAP` of 200.
- For Q&A with follow-ups, the input tokens grow with each turn as conversation history accumulates. The 6,000 estimate assumes 3–4 turns of context.
- For summarization, the 8,000 input tokens assume the user submits a moderately long section. With documents averaging 100 pages, if users attempt full-document summarization, the input could be much higher — potentially exceeding 20,000+ tokens depending on how much content is passed.
- These estimates are for the **testing phase** with 30 users. Production workloads with more users would require re-evaluation, particularly the peak calls per minute.

### 5.2 Workload Definitions

| Use case | Peak calls/min | Input tokens | Output tokens | Description |
|----------|---------------|-------------|--------------|-------------|
| RAG chat | 10 | 4,000 | 500 | Primary use case — user asks a question, orchestrator retrieves chunks from AI Search, LLM generates a grounded answer |
| Basic chat | 8 | 1,500 | 400 | No retrieval — casual conversation, clarifications, general questions without document context |
| Summarization | 3 | 8,000 | 1,000 | User submits or references a long document section for the model to summarize |
| Q&A with follow-ups | 6 | 6,000 | 600 | Multi-turn RAG conversation where chat history accumulates across turns |

---

## 6. PTU Calculator Results

All three models were evaluated using the Microsoft Foundry PTU calculator with the workload definitions from Section 5. The minimum deployment size for Data Zone Provisioned is 15 PTU with 5 PTU increments.

### 6.1 PTU Breakdown Per Workload

| Use case | Calls/min | In / Out tokens | gpt-4.1 PTU | gpt-4.1-mini PTU | gpt-5.1 PTU |
|----------|-----------|----------------|-------------|------------------|-------------|
| RAG chat | 10 | 4,000 / 500 | 20 | 15 | 20 |
| Basic chat | 8 | 1,500 / 400 | 15 | 15 | 15 |
| Summarization | 3 | 8,000 / 1,000 | 15 | 15 | 15 |
| Q&A with follow-ups | 6 | 6,000 / 600 | 20 | 15 | 15 |
| **Total PTU** | | | **100** | **100** | **100** |
| **Utilization** | | | **57.07%** | **11.49%** | **48.51%** |

### 6.2 Analysis

All three models land at **100 PTU** because the workloads for 30 test users are small enough to hit the minimum deployment floor (15 PTU per workload, rounded up to the next 5 PTU increment where needed).

The **utilization percentage** is where the real difference shows:

- **gpt-4.1 at 57%** — using over half the capacity. This is a healthy utilization for testing. There is room for traffic growth, but scaling to significantly more users would require additional PTUs.
- **gpt-4.1-mini at 11.5%** — massive headroom. The same 100 PTU could handle roughly 5x the current workload before approaching capacity limits. This model is significantly over-provisioned at this scale, meaning it would be the most cost-effective choice if we want to scale the user base without increasing PTUs.
- **gpt-5.1 at 48.5%** — sits between the other two. The slightly higher input TPM/PTU (4,750 vs 3,000) helps compared to gpt-4.1, but the 1:8 output ratio (vs 1:4) means output-heavy workloads like summarization consume relatively more capacity.

### 6.3 Key Takeaway

At the testing scale of 30 users, cost is identical across all three models (100 PTU each). The decision should therefore be driven by **quality requirements**, not cost:

- Start with **gpt-4.1** as the default — best quality-to-cost ratio, proven for RAG
- Test **gpt-4.1-mini** in parallel to evaluate if the quality difference matters for our specific document corpus. If quality is acceptable, this model provides the best scaling path
- Test **gpt-5.1** for complex queries to see if the reasoning capability and larger context window add measurable value

---

## 7. Recommendations and Next Steps

1. **Deploy gpt-4.1** on Data Zone Provisioned at the calculator-recommended PTU as the primary model. Create a matching Data Zone Standard deployment for spillover.
2. **Run A/B quality tests** with gpt-4.1-mini and gpt-5.1 using representative queries from our document corpus to evaluate answer quality differences.
3. **Monitor actual token usage** during testing to validate or adjust the workload estimates in Section 5. Pay attention to actual `top_k` chunk sizes and conversation history growth.
4. **Re-evaluate at production scale** — when moving beyond 30 test users, the utilization differences between models become the primary cost driver. gpt-4.1-mini's 5x efficiency advantage will translate to real savings at higher user counts.
5. **Consider PTU reservations** — for long-term deployment, purchasing Azure Reservations provides significant discounts over hourly billing. Deploy first to confirm capacity, then purchase a matching reservation.

---

*Prepared — April 2026*
