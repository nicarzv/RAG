# Interview Questions — Follow-up: Feature Delivery & Code Quality

**Context:** Senior software developer/engineer to extend the GPT-RAG accelerator (Python 3.12, FastAPI, Chainlit, Azure Container Apps). This person will be coding on our apps from day one.  
**Stack:** Orchestrator (FastAPI + Uvicorn, async, Semantic Kernel), Frontend (FastAPI + Chainlit 2.9.4 + React components), Ingestion (FastAPI + APScheduler). All deployed as Container Apps. Configuration via Azure App Configuration. Data in Cosmos DB + Azure AI Search. Auth via Entra ID with OBO flow.  
**Strategy in use:** `maf_lite` (calls LLM directly via Chat Completions API — NOT through Agent Service).  
**Duration:** ~60 minutes  
**Format:** 12 questions in 3 sections. All questions include evaluation rubrics.

---

## Section A — Feature Delivery (4 questions, ~25 min)

These questions test whether the candidate can take a feature from requirements to working code in our specific stack.

### Q1 — Document upload: end-to-end design

**Ask:**
Users currently rely on an admin to push documents into SharePoint, then our ingestion pipeline picks them up on a cron schedule via APScheduler and indexes them into Azure AI Search. We want to add a "Upload Document" button in the Chainlit chat UI so users can upload files directly and have them become searchable. Walk me through the full flow — what changes in the frontend, what changes in the backend, and how does the document end up in the search index?

**Follow-up probes (use during their answer):**
- "Our frontend has zero direct access to AI Search or OpenAI — it only talks to the Orchestrator. And the Orchestrator has `SearchIndexDataReader` — read-only. Only the Ingestion app has `SearchIndexDataContributor`. How does that affect your design?"
- "The ingestion pipeline uses Document Intelligence to extract text, then `langchain-text-splitters` to chunk at 2,048 tokens with 200 overlap, then generates embeddings with `text-embedding-3-large` at 3,072 dimensions. How do you make sure an ad-hoc upload goes through the same pipeline?"
- "What about permissions? If user A uploads a document, should user B be able to find it via RAG? Our search index has `metadata_security_user_ids` and `metadata_security_group_ids` fields that are checked via OBO token at query time."

**What a good answer covers:**

*Architecture decision — who handles the upload:*
The Frontend accepts the file (Chainlit supports file uploads via `@cl.on_message` with attached files or a custom action button). The file is sent to a backend endpoint. But which backend? The Orchestrator can't write to the search index (read-only RBAC). The Ingestion app can write but currently only runs on a cron schedule. The candidate must recognize this RBAC constraint and propose a solution: either (a) send the file from Frontend → Ingestion app directly via a new HTTP endpoint on Ingestion, (b) drop the file into Blob Storage and let the next cron cycle pick it up, or (c) add a new endpoint to Ingestion that triggers on-demand ingestion for a single document.

*Option (b) — drop to Blob Storage — is the simplest:*
Frontend uploads file → Blob Storage (Frontend already has `StorageBlobDataReader` + `StorageBlobDelegator`, but would need `StorageBlobDataContributor` added via Bicep). Then the existing ingestion cron picks it up on the next cycle. Pros: minimal code changes, reuses the entire pipeline. Cons: delay (cron runs on a schedule, not instantly). The candidate should mention this trade-off.

*Option (c) — trigger Ingestion on-demand — is the better answer:*
Frontend uploads to Blob Storage → sends a trigger request to the Ingestion app (new REST endpoint like `POST /ingest-document`) → Ingestion runs the full pipeline (Document Intelligence → chunking → embedding → index write) for that single document → returns status. This gives near-real-time availability. The candidate should think about: how does Frontend talk to Ingestion? Currently it only talks to Orchestrator. They'd need either a direct Container App-to-Container App call (Dapr invoke since both have Dapr enabled) or route through the Orchestrator as a proxy.

*Permissions on uploaded documents:*
When a user uploads a document, the ingestion pipeline needs to stamp the correct ACL fields (`metadata_security_user_ids`, `metadata_security_group_ids`) into the search index. For SharePoint documents, these come from Graph API. For user-uploaded documents, the candidate needs to decide: stamp only the uploader's user ID (private), stamp the uploader's group memberships (team-visible), or make it accessible to everyone. This is a product decision, but the candidate should recognize it as a question that needs answering and propose how the ACL fields would be populated from the JWT claims in the upload request.

*Error handling:*
What if Document Intelligence fails to extract text (unsupported format, corrupt file)? What if embedding generation hits a rate limit? The candidate should propose async status tracking — the user gets an immediate "uploading..." response and can check status later, rather than holding the HTTP connection open for minutes.

**What separates good from great:**
A great candidate immediately spots the RBAC issue without being prompted. They ask about file size limits (Container Apps have a default 4MB request body limit — you'd need to change `max_request_body_size` in the Container App config or use chunked/multipart upload). They consider duplicate detection — what if the user uploads the same file twice? Check by hash or filename before re-ingesting. They mention that Chainlit 2.9.4 has native file upload support via `cl.AskFileMessage` or file attachments on `@cl.on_message`, so the UI work is minimal.

---

### Q2 — Domain-specific chat: filtering and prompt scoping

**Ask:**
We want users to select a "domain" when chatting — like HR, Legal, or Finance — and the chatbot should only search and answer from documents in that domain. Looking at our current search flow: the Orchestrator sends the user query to Azure AI Search using hybrid search (BM25 + vector), gets back the top 3 chunks (`SEARCH_RAGINDEX_TOP_K=3`), and passes them to the LLM as context. What do you need to change in the ingestion pipeline, the search query, the orchestrator, and the frontend?

**Follow-up probes:**
- "Our search connector (`connectors/search.py`) constructs the search query. It already applies a permission filter via OBO token. Where does the domain filter go — in the same query or a separate step?"
- "Should the system prompt change per domain? Right now it's loaded from `prompts/single_agent_rag/main.txt` at startup and reused for every request."
- "What if a user asks a question that doesn't belong in the selected domain — like asking an HR question while in the Finance domain?"

**What a good answer covers:**

*Ingestion changes:*
Add a `domain` metadata field to each chunk at ingestion time. This must be populated during the chunking step in the ingestion pipeline — either from the source folder structure (e.g. SharePoint library name), a mapping file, or a classification step. The field needs to be defined in the AI Search index schema as a `filterable` string field. If it's not filterable, you can't use it in an OData `$filter` expression. The candidate should know this distinction.

*Search query changes:*
In `connectors/search.py`, the search request needs an additional OData filter: `domain eq 'HR'`. This is combined with the existing permission filter using `$filter=domain eq 'HR' and (metadata_security_group_ids/any(g: g eq '...') or ...)`. The candidate should know that Azure AI Search OData filters use `and`/`or` logical operators and `any()` for collection fields.

*Orchestrator changes:*
The `POST /orchestrator` request payload currently has `conversation_id`, `question`, and `question_id`. Add a `domain` field. The orchestrator passes this to the search connector for filtering. Additionally, switch `PROMPT_SOURCE` from `file` to `cosmos` (or use a template), so you can store domain-specific system prompts — `single_agent_rag_main_hr`, `single_agent_rag_main_legal`, etc. — and load the right one based on the domain parameter. Currently the prompt is loaded once at startup and baked into the persistent agent, so domain-specific prompts require either creating separate agents per domain or recreating the agent per request (which is expensive).

*Frontend changes:*
Add a domain selector to the Chainlit UI. Chainlit supports custom React components (the codebase already has `FeedbackForm.jsx` in `public/elements/`). Create a domain picker component, persist the selection in the Chainlit session (`cl.user_session`), and include it in every request to the Orchestrator.

*Cross-domain queries:*
Options: (a) reject and tell the user their question seems outside the current domain, (b) search across all domains and flag which domain the answer came from, (c) use an intent classifier (cheap LLM call or keyword match) to detect domain mismatch and suggest switching. The candidate should pick one and justify it.

**What separates good from great:**
A great candidate thinks about the AI Search index schema change as a migration problem — existing chunks don't have a `domain` field, so you need to either re-index everything (expensive) or add the field with a default value and backfill. They mention that making the domain field `filterable` and `facetable` allows them to also show a count of documents per domain in the UI. They question whether 3 domains is enough or if this should be a generalized "collection" or "tag" system that scales. They also note that with domain filtering, `SEARCH_RAGINDEX_TOP_K=3` might need to increase since the search space is now narrower — top 3 from 4,000 documents vs top 3 from maybe 500 HR documents (higher chance the right chunks are in the top 3 already, but worth testing).

---

### Q3 — Chat resume: conversation persistence with context management

**Ask:**
When a user closes the browser and reopens it, their conversation history is gone. We want to show a sidebar with previous conversations that users can click to resume. The conversation data is actually already saved in Cosmos DB — each conversation has an `id`, a `questions` array, and a `thread_id` for the AI Agents thread. But the frontend doesn't load previous conversations — it always starts fresh. What do you need to build?

**Follow-up probes:**
- "The Orchestrator streams responses via SSE — the first chunk is the `conversation_id`, then answer tokens, then `TERMINATE`. But the SSE stream doesn't include the assistant's response in the Cosmos document — only the user's questions are stored in `questions[]`. Where do the assistant answers come from when loading a previous conversation?"
- "If a conversation has 50 messages, how do you handle the context window? The Orchestrator currently uses `tiktoken` to count tokens and truncates conversation history (oldest first) to fit. Is that enough for resumed conversations?"
- "Chainlit has built-in data persistence. Should you use Chainlit's persistence layer or our custom Cosmos DB solution?"

**What a good answer covers:**

*The missing piece — assistant responses are NOT in Cosmos:*
This is the critical insight. The current Cosmos document only stores `questions[]` (user messages) and `thread_id`. The assistant's responses are in the AI Agents thread (accessed via `thread_id`), not in our Cosmos DB. To display a previous conversation, the candidate needs to either: (a) fetch the thread from the AI Agents API to get both user and assistant messages, or (b) start storing assistant responses in Cosmos alongside the questions. Option (b) is more reliable — the Agents thread may be garbage-collected or unavailable, while Cosmos data is under our control. This means modifying `orchestrator.py` to also append the assistant's response to the conversation document after streaming completes.

*Frontend — Chainlit session management:*
Chainlit 2.9.4 has a built-in data persistence layer (`@cl.data_layer`) but the accelerator doesn't use it — it has its own custom Cosmos persistence in the Orchestrator. The candidate should recommend using the existing Cosmos approach (consistency, single source of truth) and building a "Load conversation" API endpoint on the Orchestrator that returns the full message history. The frontend calls this on `@cl.on_chat_start` and can render a conversation list using Chainlit's `cl.Message` replay or a custom React sidebar component.

*New API endpoints needed:*
- `GET /conversations?user_id=xxx` — list previous conversations (with title/preview) for the authenticated user
- `GET /conversations/{id}` — load full message history
- Both secured with the same JWT validation as `/orchestrator`
- The Orchestrator already has Cosmos DB access (`CosmosDBBuiltInDataContributor`) so it can query conversations by user_id

*Auto-generated titles:*
Show meaningful titles in the sidebar, not UUIDs. Option: after the first Q&A exchange, make a cheap LLM call (or use the first 50 characters of the first question) to generate a title. Store it in the Cosmos conversation document.

*Context window on resume:*
For long conversations, the existing truncation logic (oldest messages first via tiktoken) works, but the candidate should note that truncating is lossy — the user can see all 50 messages in the sidebar, but the LLM only gets the last N. They should propose either: showing the user that older messages are "not in active context" (transparency), or using a summarization step to compress older messages into a summary block that preserves key context. They should reference that gpt-5.1 has 400K context, so this is less urgent than with smaller models, but still matters for very long conversations.

**What separates good from great:**
A great candidate immediately asks: "Are assistant responses stored anywhere?" — this is the key architectural gap. They think about data retention and GDPR — storing conversation history in Sweden-Central Cosmos DB aligns with EU data residency, but they should ask about retention policies and user-initiated deletion. They consider what happens when the user tries to resume a conversation but the backend deployment has changed (e.g. different system prompt) — the context may be stale. They propose a `last_active` timestamp on conversation documents and a TTL policy in Cosmos to auto-expire old conversations.

---

### Q4 — Content safety: graceful handling of Azure OpenAI guardrails

**Ask:**
Azure OpenAI has built-in content safety filters. Sometimes a legitimate user query triggers a false positive — the model refuses to answer. Currently our app shows "Something went wrong" because the error is caught as a generic exception. I want you to look at this from two angles: first, how do you detect that this specific error happened versus a real error, and second, what should the user experience be?

**Follow-up probes:**
- "The Azure OpenAI API response includes a `content_filter_results` object and the completion can have `finish_reason: 'content_filter'`. How do you distinguish between the input being filtered (user's query was flagged) versus the output being filtered (model's generated response was flagged)?"
- "Should we retry? Under what circumstances?"
- "We're required to have audit logging for compliance. What do you log?"

**What a good answer covers:**

*Detection — structured error parsing:*
The Azure OpenAI Python SDK (`openai` package) raises a `BadRequestError` with a specific error code when the input is filtered, and the streaming response may include `finish_reason: "content_filter"` when the output is filtered. The candidate should know these are two different cases:

- **Input filtered:** The API rejects the request before generating anything. The error response includes a `content_filter_result` with `filtered: true` for one of the categories (hate, sexual, violence, self_harm). Status code 400.
- **Output filtered:** The model starts generating but the output hits a filter mid-stream. The SSE stream may end abruptly with `finish_reason: "content_filter"`. The user might see a partial response followed by nothing.

*User experience:*
For input filtered: "I wasn't able to process that question due to content safety policies. Try rephrasing your question." Don't expose which category triggered (that leaks filter thresholds). Don't show the generic "Something went wrong" — the user will think the app is broken and retry the same query.

For output filtered: trickier because the user already saw partial tokens streaming. Options: (a) append a message like "I couldn't complete this response due to content safety policies," (b) clear the partial response and show the error message. The candidate should prefer (a) since clearing streamed content is a bad UX.

*Retry logic:*
Input filtered: NO retry — same input will hit the same filter. Output filtered: MAYBE retry once with a modified system prompt that includes stronger guardrails like "Do not generate content that could be flagged by content safety filters. If you're unsure about a response, provide a conservative answer." But this adds latency and may still fail. The candidate should default to no retry and instead focus on good error messaging.

*Audit logging:*
Log: timestamp, user ID, conversation ID, the original query (or a hash if PII concerns), which filter category triggered, severity level, input vs output, and the action taken (blocked, partial response). This goes to Application Insights via the existing OpenTelemetry integration. The candidate should mention that Azure OpenAI also has its own content filtering logs in Azure Monitor — so there are two sources of truth. For compliance, log at least in your own telemetry so you're not dependent on Azure's retention policies.

*Code-level implementation:*
In the maf_lite strategy, wrap the `chat.completions.create()` call with specific exception handling: catch `openai.BadRequestError`, check for content filter error codes, and return a structured error to the SSE stream instead of propagating a 500. For output filtering, check each streamed chunk's `finish_reason` and handle `content_filter` gracefully.

**What separates good from great:**
A great candidate thinks about the RAG-specific case: the user's query is innocent, but one of the 3 retrieved chunks contains sensitive content (e.g. a legal document about sexual harassment policy), and the combination triggers the output filter. The fix isn't on the user — it's a data issue. They propose: log which chunks were in context when the filter triggered, so the team can review whether certain documents need to be excluded or the filter thresholds adjusted. They also mention Azure OpenAI's configurable content filter — you can create a custom content filter configuration with adjusted severity thresholds per category — and that this is an Azure portal setting, not a code change. They know this exists but defer the threshold decision to the business.

---

## Section B — Code Quality & Refactoring (4 questions, ~20 min)

### Q5 — Removing unused strategies from the orchestrator

**Ask:**
Our orchestrator uses the strategy pattern — `agent_strategy_factory.py` selects the active strategy from `AGENT_STRATEGY` in App Configuration. We use `maf_lite`. The codebase also has: `single_agent_rag_strategy_v2.py` (Azure AI Agents API), `single_agent_rag_strategy_v1.py` (legacy), `mcp_strategy.py` (Semantic Kernel + MCP), and `nl2sql_strategy.py` (3-agent group chat with SQL). These pull in heavy dependencies — `semantic-kernel`, `azure-ai-agents`, `azure-ai-projects`, `pyodbc`, `msgraph-sdk`. We want to strip out everything except maf_lite. How do you approach this?

**Follow-up probes:**
- "The Dockerfile includes `ODBC Driver 18 for SQL Server` in its system dependencies — that's only needed by NL2SQL. How do you find all such hidden dependencies?"
- "The `base_agent_strategy.py` has shared logic — prompt loading, credential management, Cosmos DB conversation handling. maf_lite inherits from it. How do you make sure you don't break the base class while removing its other children?"
- "We forked this from the Microsoft accelerator at github.com/Azure/gpt-rag-orchestrator. We may want to pull upstream updates in the future. How does that affect your refactoring approach?"

**What a good answer covers:**

*Dependency mapping:*
Before deleting any file, trace the full dependency tree. Start from the files to remove (`mcp_strategy.py`, `nl2sql_strategy.py`, `single_agent_rag_strategy_v1.py`, `single_agent_rag_strategy_v2.py`) and trace:
- **Python imports:** What do these files import that nothing else uses? Use `pylint` unused imports, `vulture` for dead code, or IDE "find usages." The `plugins/` directory (`retrieval/plugin.py`, `nl2sql/plugin.py`, `common/plugin.py`) is likely only used by MCP and NL2SQL strategies.
- **pip dependencies:** `semantic-kernel`, `azure-ai-agents`, `azure-ai-projects`, `pyodbc` — check if maf_lite uses any of these. If not, remove from `requirements.txt`.
- **System dependencies:** ODBC Driver 18 in the Dockerfile is only for `pyodbc`/NL2SQL. Removing it shrinks the Docker image.
- **Configuration keys:** `MCP_SERVER_URL`, `DATABASE_ACCOUNT_NAME`, `DATASOURCES_DATABASE_CONTAINER` — only used by removed strategies. Clean up App Configuration.
- **Prompt files:** `prompts/mcp/main.txt`, `prompts/nl2sql/*.txt` — can be deleted.
- **Cosmos containers:** `datasources` container is for NL2SQL datasource configs — may not be needed anymore.

*Incremental removal order:*
1. First, write or verify integration tests that exercise the maf_lite path end-to-end (POST /orchestrator → search → LLM → SSE response).
2. Remove NL2SQL strategy + its plugins + SQL dependencies + ODBC driver. Test. Commit.
3. Remove MCP strategy + Semantic Kernel dependency. Test. Commit.
4. Remove single_agent_rag_v1 and v2 + `azure-ai-agents`/`azure-ai-projects` dependencies. Test. Commit.
5. Clean up the factory: remove the `if/elif` branches, or simplify to always return maf_lite.
6. Clean up `base_agent_strategy.py`: remove any methods only used by the deleted strategies.
7. Clean up `requirements.txt`, Dockerfile, App Configuration keys, prompt files.

*Base class safety:*
The candidate should read `base_agent_strategy.py` carefully before deleting strategies. It has shared logic (prompt loading via `_read_prompt()`, credential management via `IdentityManager`, Cosmos persistence). After removing child strategies, some methods in the base class may become dead code. But they should NOT delete methods that maf_lite still uses — trace call chains from maf_lite up to the base class.

*Upstream merge strategy:*
This is the hard question. If they delete files that exist upstream, every `git merge` from the Microsoft repo will have conflicts on those files. Options:
- **Accept the merge cost:** Maintain a clear list of what was removed and why. When merging upstream, resolve conflicts by re-deleting the files. This is manageable if upstream doesn't change these files often.
- **Feature flag instead of delete:** Keep the strategy files but make them unreachable (remove from factory, keep `AGENT_STRATEGY=maf_lite` only). This avoids merge conflicts but leaves dead code. The candidate should argue against this — the whole point is reducing complexity.
- **Git subtree/patch approach:** Maintain the fork with clear divergence tracking. Document the removal in a `FORK_CHANGES.md` so future developers know what changed.

**What separates good from great:**
A great candidate asks: "What does removing these strategies actually gain us beyond cleaner code?" and answers their own question: faster Docker builds (smaller image without ODBC), fewer pip dependencies (reduced supply chain attack surface), simpler onboarding (new developers don't need to understand 4 strategies), and fewer App Configuration keys to manage. They estimate the effort: this is probably 1-2 days of careful work, not 2 weeks — the actual deletion is fast, the dependency tracing and testing is the work. They mention running `pip-audit` or `safety check` after removing dependencies to verify no new vulnerabilities were introduced.

---

### Q6 — Implementing concurrency limits in an async Python orchestrator

**Ask:**
Our orchestrator is a FastAPI + Uvicorn app, fully async — all handlers are `async def`, and it uses `aiohttp` for AI Search and `AsyncAzureOpenAI` for LLM calls. Currently there are zero concurrency controls — if 100 users send requests simultaneously, all 100 fire outbound calls to Azure OpenAI at the same time. Our gpt-5.1 deployment has provisioned throughput at 25-50 PTU with spillover to Data Zone Standard. What happens under load, and how do you fix it?

**Follow-up probes:**
- "The orchestrator runs on a single Container App replica with 0.5 vCPU and 1 GiB RAM. How does that affect your concurrency design?"
- "Our maf_lite strategy makes two LLM calls per request: first a short intent classification call (~200 max_completion_tokens), then the main response stream. Should these share the same semaphore?"
- "The `aiohttp` session for AI Search has no explicit request timeout — it uses the aiohttp default of 300 seconds. What's the risk?"

**What a good answer covers:**

*What happens without limits:*
The provisioned deployment handles requests up to its PTU capacity, then excess spills to Data Zone Standard (pay-per-use). If Standard is also overwhelmed, requests get HTTP 429 with `retry-after-ms` headers. With no concurrency control, all 100 requests fire simultaneously — the first batch succeeds, the rest get 429s. If the app doesn't handle 429s (and currently, only `get_embeddings()` handles them — Search and LLM calls don't), the user sees errors. Worse, if all 429 retries fire at the same moment, you get thundering herd.

*Semaphore for outbound LLM calls:*
```python
import asyncio

llm_semaphore = asyncio.Semaphore(20)  # limit concurrent LLM calls

async def call_llm(prompt):
    async with llm_semaphore:
        return await openai_client.chat.completions.create(...)
```
The candidate should explain why `asyncio.Semaphore` works here: the app is single-threaded async, so the semaphore is just a coroutine-level primitive — no thread safety concerns. Requests beyond the limit wait in a FIFO queue until a slot opens. They should discuss how to choose the limit: estimate from the provisioned PTU capacity. With gpt-5.1 at 4,750 input TPM per PTU, 50 PTU = ~237,500 input tokens/minute. If average request is ~6,000 input tokens, that's ~39 requests/minute, or roughly 5-10 concurrent requests before spillover starts. Set the semaphore conservatively at 10-15.

*Separate semaphores for separate services:*
AI Search and OpenAI have different rate limits and different characteristics. Search is fast (sub-second), OpenAI is slow (streaming, 10-20s). Use separate semaphores:
```python
search_semaphore = asyncio.Semaphore(30)   # Search is fast, can handle more
llm_semaphore = asyncio.Semaphore(15)       # LLM is slower, fewer concurrent
```
The candidate should NOT use a single global semaphore for both — that would make LLM calls block Search calls unnecessarily.

*Intent classification vs main response:*
The maf_lite strategy makes two LLM calls: first a small classification call (~200 tokens), then the main response (potentially thousands of tokens). Options: (a) share the same semaphore (simpler, but the small call gets queued behind big ones unnecessarily), (b) separate semaphores or weighted approach (classification call uses 1 slot, main call uses N slots proportional to expected token usage). A pragmatic answer is to share the semaphore for simplicity — the classification call is so fast it'll release the slot quickly.

*429 handling with exponential backoff:*
```python
async def call_llm_with_retry(prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            async with llm_semaphore:
                return await openai_client.chat.completions.create(...)
        except openai.RateLimitError as e:
            retry_after = e.response.headers.get("retry-after-ms", 1000)
            jitter = random.uniform(0, float(retry_after) * 0.5)
            await asyncio.sleep((float(retry_after) + jitter) / 1000)
    raise ServiceUnavailableError("LLM unavailable after retries")
```
Key points: read `retry-after-ms` from Azure's response header (not hardcoded). Add jitter to prevent thundering herd. Limit retries (3 max). Return a user-friendly error if all retries fail.

*Timeout on AI Search:*
The aiohttp session with 300s default timeout is a time bomb. If AI Search becomes slow, requests hang for 5 minutes before timing out. Set explicit timeouts:
```python
session = aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=5))  # 5s for search
```
The candidate should set per-service timeouts: Search = 5s (it should be fast), LLM = 30s (streaming takes time), Cosmos DB = 5s (reads are fast).

**What separates good from great:**
A great candidate discusses observability: add metrics for semaphore queue depth (how many requests are waiting), 429 count per service, and average wait time. These go to Application Insights via the existing OpenTelemetry setup. They can create alerts: "if LLM semaphore queue > 10 for 60 seconds, consider adding PTUs." They also mention that with only 0.5 vCPU and 1 GiB RAM on the Container App, even if you handle concurrency perfectly, you'll hit CPU and memory limits. JSON serialization, tiktoken token counting, and prompt construction are CPU-bound — they block the event loop. They suggest increasing the Container App resources or adding replicas (currently min:1, max:1 — change to max:3 with autoscaling rules based on HTTP concurrency).

---

### Q7 — Testability: refactoring tightly coupled Azure clients

**Ask:**
Our orchestrator creates all its Azure clients in the FastAPI `lifespan()` hook — `SearchClient`, `AsyncAzureOpenAI`, `CosmosDBClient`, `IdentityManager` — as singletons that are reused across all requests. This is great for production performance. But it means you can't write a unit test for the maf_lite strategy without having real Azure credentials, a real AI Search index, and a real OpenAI deployment. How would you refactor this so the strategy logic is testable without hitting Azure services?

**Follow-up probes:**
- "Show me what the maf_lite strategy constructor would look like after refactoring. What parameters does it receive?"
- "The `IdentityManager` is a singleton that caches tokens using `ChainedTokenCredential` — `ManagedIdentityCredential` in production, `AzureCliCredential` for local dev. How do you handle this in tests?"
- "The search connector constructs an OBO token exchange for permission-filtered queries. How do you test the permission filtering logic without a real Entra ID tenant?"

**What a good answer covers:**

*Dependency injection via constructor:*
Instead of the strategy creating or accessing global clients, inject them:

```python
# Before (tightly coupled):
class MafLiteStrategy(BaseAgentStrategy):
    def __init__(self, config):
        self.search_client = SearchClient(...)  # created internally
        self.openai_client = AsyncAzureOpenAI(...)  # created internally

# After (injectable):
class MafLiteStrategy(BaseAgentStrategy):
    def __init__(self, config, search_client, openai_client, cosmos_client):
        self.search_client = search_client
        self.openai_client = openai_client
        self.cosmos_client = cosmos_client
```

The lifespan hook still creates the real clients and passes them to the strategy. But in tests, you pass mocks.

*Python Protocols for type safety:*
Define what the strategy actually needs from each client:

```python
from typing import Protocol

class SearchService(Protocol):
    async def search(self, query: str, top_k: int, user_token: str | None) -> list[dict]: ...

class LLMService(Protocol):
    async def complete(self, messages: list, **kwargs) -> AsyncIterator[str]: ...
```

The strategy depends on the Protocol, not the concrete Azure SDK class. Tests implement the Protocol with stubs that return predictable data. This also documents the exact interface the strategy uses — new developers can see what methods matter.

*Testing the OBO flow:*
The permission filtering logic has two parts: (1) the OBO token exchange (calls Entra ID), and (2) passing the token to AI Search as `x-ms-query-source-authorization`. For unit tests, mock the OBO exchange to return a fake token, and verify that the search connector passes it correctly. For integration tests of the actual permission filtering, you need a real Entra tenant — this is an integration test, not a unit test. The candidate should draw this boundary clearly.

*Recorded HTTP interactions:*
For a middle ground between mocks and live services, use `vcrpy` or `responses` library to record real HTTP interactions once, then replay them in tests. This captures the actual response shapes and edge cases (error codes, rate limits, empty results) without needing live credentials on every test run.

**What separates good from great:**
A great candidate preserves the singleton pattern for production — the refactoring doesn't change the production architecture (clients are still created once at startup and reused). It only adds an injection seam for testing. They mention that the current `base_agent_strategy.py` likely has some methods that mix business logic with Azure SDK calls — those methods need to be split. They propose a test structure: `tests/unit/` (mocks, fast, runs in CI), `tests/integration/` (real Azure, runs manually or nightly). They estimate effort: refactoring for injectable dependencies is probably 2-3 days, writing the first round of unit tests is another 2-3 days.

---

### Q8 — Designing error handling and user feedback for a streaming API

**Ask:**
Our orchestrator streams responses via Server-Sent Events. The SSE format is: first chunk = conversation_id, then answer tokens, then `TERMINATE`. If an error occurs mid-stream — say Azure OpenAI times out after streaming 50 tokens — the user sees a partial answer and then... nothing. The stream just stops. No error message, no indication of what happened. How would you design error handling for a streaming API where the response has already started?

**Follow-up probes:**
- "What if the error happens before any tokens are streamed — is that easier or harder to handle?"
- "The frontend (`orchestrator_client.py`) reads the SSE stream using `httpx`. How does it know the stream ended abnormally versus normally?"
- "Should errors during streaming be saved to the Cosmos conversation document? What does the user see when they resume that conversation?"

**What a good answer covers:**

*Error classification by timing:*
- **Before streaming starts** (search fails, auth fails, prompt construction fails): Return an HTTP error response (4xx or 5xx) — the frontend hasn't started rendering anything yet, so it can show a clean error message. This is straightforward.
- **During streaming** (LLM timeout, content filter, network interruption): The HTTP response is already 200 with `Content-Type: text/event-stream`. You can't change the status code. You need an in-band error signal.

*In-band error protocol:*
Define a structured error event in the SSE stream:
```
data: Based on the company policy
data: , employees are entitled to
data: ERROR:{"code":"llm_timeout","message":"The response could not be completed. Please try again."}
data: TERMINATE
```
The frontend parser checks each chunk: if it starts with `ERROR:`, parse the JSON and display the error message instead of rendering it as answer text. Always send `TERMINATE` after the error so the frontend knows the stream is done — otherwise it hangs waiting for more data.

*Frontend handling (`orchestrator_client.py`):*
The httpx SSE reader in `orchestrator_client.py` currently buffers tokens and renders them. Add a check: if a chunk matches the error pattern, stop rendering answer tokens, display the error to the user via `cl.ErrorMessage` or `cl.Message` with an error indicator. If the stream ends without `TERMINATE` (e.g. TCP connection dropped), the httpx reader raises an exception — catch this and show "Connection lost, please try again."

*Partial response handling:*
If the user saw 50 tokens of a valid answer before the error, do you keep them? Options: (a) keep the partial answer and append an error notice — "⚠️ This response was interrupted. The information above may be incomplete." (b) discard and show only the error. Option (a) is better UX — the partial answer might still be useful. Option (b) is safer — a partial answer could be misleading if it stops mid-sentence.

*Cosmos persistence:*
If an error occurs mid-stream, the conversation should be saved with a flag indicating the response was incomplete. Propose adding a `status` field to the conversation question entry: `"complete"`, `"partial"`, `"error"`. When the user resumes this conversation, the frontend can show that the last response was incomplete and offer a "Retry" button.

*Timeout budget:*
The candidate should connect to Q6: set a total request timeout (e.g. 30s). If the LLM hasn't finished streaming by 30s, send the error event and close the stream. Don't let the user wait indefinitely. The timeout should be measured from the first token — not from when the request was sent (since there may be queuing time).

**What separates good from great:**
A great candidate proposes a heartbeat mechanism: if the LLM is thinking and no tokens have been emitted for 10 seconds, send a `data: \n\n` (empty SSE comment/ping) to keep the HTTP connection alive and let the frontend know the backend is still working. This prevents load balancers and proxies (Container Apps ingress has a default idle timeout of 240s, but intermediaries may be shorter) from dropping the connection. They also consider retry from the user's perspective — if the user hits "Retry," should the orchestrator re-run the search (chunks may have changed) or reuse the cached search results from the failed attempt? They suggest caching the search results in the conversation for retry purposes.

---

## Section C — Architecture & Algorithmic Thinking (4 questions, ~15 min)

### Q9 — Designing a multi-model routing strategy

**Ask:**
We have three model deployments: gpt-5.1 (recommended, provisioned at 25-50 PTU + spillover), gpt-4.1 (baseline fallback, provisioned), and gpt-4.1-mini (cost efficient, provisioned). Currently the orchestrator always uses whatever model is in `CHAT_DEPLOYMENT_NAME` in App Configuration. We want smarter routing — some requests go to gpt-5.1, some to gpt-4.1-mini, and if a model is overloaded, we fall back automatically. Design the routing logic.

**Follow-up probes:**
- "gpt-5.1 has a 1:8 output token ratio, gpt-4.1 has 1:4. For a summarization request that generates 1,000 output tokens, which model is more cost-efficient on provisioned throughput and why?"
- "How do you detect 'overloaded' when we have spillover? A 429 from the provisioned deployment means spillover is also saturated."
- "Should routing decisions be per-request or can you batch? What about the intent classification call that maf_lite makes before the main response?"

**What a good answer covers:**

*Workload-based routing:*
Route based on the type of request:
- **RAG chat** (primary use case) → gpt-5.1 — best quality for grounded answers, configurable reasoning
- **Basic chat** (no retrieval) → gpt-4.1-mini — doesn't need top-tier quality for casual conversation
- **Summarization** (high output) → gpt-4.1 — with 1:4 output ratio, 1,000 output tokens = 4,000 input token equivalent. On gpt-5.1 with 1:8, the same 1,000 tokens = 8,000 input token equivalent. gpt-4.1 is 2x cheaper for output-heavy workloads.
- **Q&A with follow-ups** → gpt-5.1 — multi-turn needs the best instruction following

The routing decision depends on detecting the request type. The intent classification call that maf_lite already makes could be extended to also classify the workload type.

*Fallback chain:*
```
gpt-5.1 (primary) → gpt-4.1 (fallback) → gpt-4.1-mini (last resort)
```
On 429 from gpt-5.1: check `retry-after-ms`. If > 5 seconds, skip to gpt-4.1 immediately (spillover is saturated). If < 2 seconds, retry once (spillover might have capacity). If gpt-4.1 also 429s, fall back to gpt-4.1-mini. Combined with circuit breaker: after 3 consecutive 429s from gpt-5.1 in 60 seconds, route directly to gpt-4.1 for 30 seconds without trying gpt-5.1.

*Configuration:*
Store routing rules in App Configuration so they can be changed without redeploying:
```json
{
  "ROUTING_RULES": {
    "rag_chat": "gpt-5.1",
    "basic_chat": "gpt-4.1-mini",
    "summarization": "gpt-4.1",
    "fallback_chain": ["gpt-5.1", "gpt-4.1", "gpt-4.1-mini"]
  }
}
```

*Implementation in maf_lite:*
Add a model resolver step between the intent classification and the main LLM call. The intent classification already runs on a fixed model — this stays the same (cheap, fast, always gpt-4.1-mini). The model resolver takes the classified intent + routing rules and selects the deployment name for the main call.

**What separates good from great:**
A great candidate considers: what if gpt-4.1-mini gives a noticeably worse answer than gpt-5.1? The user doesn't know they got a fallback model. They propose adding a subtle indicator ("Response generated with a faster model due to high demand") so the user can retry later for a premium response. They think about A/B testing: route 10% of RAG chat traffic to gpt-4.1-mini and compare answer quality via the existing feedback mechanism (thumbs up/down). This gives real data for the quality vs cost trade-off. They mention that `reasoning_effort` on gpt-5.1 should be set to `none` for cost efficiency (we discussed this for our deployment) and that routing to gpt-5.1 with `reasoning_effort=medium` should be reserved for cases where the user explicitly asks for deeper analysis.

---

### Q10 — Algorithm: efficient token-aware conversation truncation

**Ask:**
The orchestrator uses `tiktoken` to count tokens in the conversation history. Before sending the prompt to the LLM, it checks if the total (system prompt + retrieved chunks + conversation history + user query) fits within the model's context window. If it doesn't, it truncates conversation history by removing the oldest messages first. This works but has a problem: what if the oldest message is actually the most important — for example, the user specified constraints in their first message like "I'm asking about policies for the Stockholm office only"? Design a better algorithm.

**What a good answer covers:**

*The naive approach's failure mode:*
Oldest-first truncation assumes recent messages are always more important. But in RAG conversations, the first message often sets the context (department, office, time period, role). Removing it changes the model's understanding of the entire conversation.

*Importance scoring:*
Score each message before deciding what to truncate:
- **High importance:** Messages with specific constraints or filters ("Stockholm office", "only for 2025", "I'm a manager"), messages that contain cited answers (they establish facts the conversation builds on), messages where the user corrects the model ("No, I meant the OTHER policy").
- **Low importance:** Short confirmations ("ok", "thanks", "got it"), repeated questions, small talk.

Implementation: a lightweight scoring function (keyword/regex-based, not an LLM call — that would be too expensive per request) that assigns a score to each message. Truncate lowest-scored messages first, regardless of age. Always keep the first message (context-setting) and the last N messages (recent context).

*Sliding window + summary hybrid:*
Instead of discarding old messages, summarize them:
1. Keep the first message (always — it sets the conversation context)
2. Keep the last 5-10 messages verbatim (recent context)
3. Everything in between: summarize into a compact "conversation so far" block
4. The summary is generated once when the history exceeds a threshold, cached in the Cosmos conversation document, and reused until new messages push past the threshold again

Token budget: System prompt (~300) + retrieved chunks (~6,000) + summary (~1,000) + recent messages (~3,000) + user query (~100) = ~10,400 tokens. With gpt-5.1's 400K context this is plenty, but with gpt-4.1's 300K it also works — truncation only matters when conversations get very long.

*When to trigger summarization:*
Don't summarize on every message — that's an extra LLM call. Trigger when: `total_history_tokens > 0.5 * context_window`. Cache the summary. Re-summarize only when the cached summary + new messages exceed the threshold again.

**What separates good from great:**
A great candidate connects this to chat resume (Q3) — if you're summarizing old messages anyway, store the summary in Cosmos. When the user resumes, load the summary + last N messages instead of all 50+ messages. They mention that tiktoken's encoding is model-specific — if you switch models mid-conversation (routing from Q9), the token counts change and the truncation threshold needs to use the target model's encoding. They propose pre-computing token counts at write time (when the response is saved to Cosmos) to avoid recounting on every request.

---

### Q11 — System design: making the ingestion pipeline trigger-based

**Ask:**
Currently the ingestion pipeline runs on a cron schedule — APScheduler triggers a job every N minutes that checks for new or modified documents. For the document upload feature (Q1), we need near-real-time ingestion — when a user uploads a file, it should be searchable within 1-2 minutes, not whenever the next cron fires. Design an event-driven ingestion trigger that coexists with the existing cron-based pipeline.

**What a good answer covers:**

*Event source:*
When a file is uploaded to Blob Storage, Azure generates an `Microsoft.Storage.BlobCreated` event. Options to capture it:
- **Event Grid** → triggers the ingestion app directly (webhook or Container App)
- **Storage Queue** → the ingestion app polls a queue (it already uses `StorageBlobDataContributor`)
- **Dapr binding** → since both Container Apps have Dapr enabled, use a Dapr input binding on blob storage events

The simplest option: Event Grid → calls a new `POST /ingest-single` endpoint on the ingestion Container App. The endpoint receives the blob path, runs the full pipeline (extract → chunk → embed → index) for that one document, and returns the status.

*Coexistence with cron:*
The cron pipeline handles bulk operations (full re-sync, delta updates from SharePoint). The event-driven trigger handles individual file uploads. They must not conflict — if the cron job picks up the same file the event trigger is already processing, you get duplicate chunks in the index. Solutions: (a) use a lock in Cosmos DB (write a "processing" status for the document ID before starting, check it before processing), (b) use the ingestion app's existing `run_id` tracking to detect and skip already-processed files.

*Error handling:*
If the event-triggered ingestion fails (Document Intelligence timeout, embedding rate limit), the event should be retried. Event Grid has built-in retry with exponential backoff. Dead-letter failed events so an admin can investigate. The user should see a status update in the chat UI ("Your document failed to process — please try again or contact support").

*Concurrency with the cron job:*
The ingestion app runs on a single replica (min:1, max:1) with 0.5 vCPU. If a cron job is processing 100 documents and an event trigger fires, the event handler runs in the same process. If the cron job holds locks or saturates the embedding rate limit, the event handler waits. The candidate should propose either queuing the event (process after cron finishes) or prioritizing event-triggered files (interrupt the batch with a priority queue).

**What separates good from great:**
A great candidate proposes a progressive approach: Phase 1 — add the `POST /ingest-single` endpoint, call it from the orchestrator after file upload, accept the latency of synchronous processing. Phase 2 — add Event Grid for true event-driven processing with retries and dead-lettering. Phase 3 — if upload volume grows, scale the ingestion Container App to multiple replicas with queue-based load distribution. They don't over-engineer the initial solution. They mention that the ingestion app already has an admin dashboard (`/dashboard`) that shows job status — extend it to show event-triggered jobs alongside cron jobs so the admin has full visibility.

---

### Q12 — Code design: implementing a clean plugin/extension point for custom features

**Ask:**
After removing unused strategies and adding new features (document upload, domain chat, chat resume), the orchestrator is growing. We keep adding `if/elif` branches and new endpoints. You're the senior developer — how would you restructure the codebase so new features can be added without modifying existing code? Think about this as: a year from now, another developer needs to add a "document comparison" feature. What should the codebase look like so they can add it cleanly?

**What a good answer covers:**

*Current problems:*
The strategy pattern is a good start — it separates orchestration logic by strategy type. But new features (document upload, domain filtering, chat resume) are cross-cutting — they apply regardless of strategy. Adding them as `if/elif` branches inside the request handler creates a god function that does everything.

*Middleware/pipeline pattern:*
Structure the request processing as a pipeline of composable steps:
```python
# Each step is a self-contained unit:
pipeline = [
    AuthenticationStep(),        # Validate JWT, extract user identity
    ConversationLoadStep(),      # Load or create conversation from Cosmos
    DomainFilterStep(),          # Apply domain filtering if selected
    SearchStep(),                # Execute AI Search with filters
    LLMCompletionStep(),         # Call the model with context
    ConversationSaveStep(),      # Save response to Cosmos
    StreamResponseStep(),        # Stream SSE to frontend
]
```
Each step receives a context object and either modifies it and passes to the next step, or short-circuits (e.g. if auth fails, skip everything and return an error).

*Feature as a module:*
A new feature like "document comparison" becomes a new step or a new endpoint module:
```
features/
├── document_upload/
│   ├── endpoint.py      # POST /upload
│   ├── handler.py       # Business logic
│   └── tests/           # Unit tests
├── domain_chat/
│   ├── filter.py        # Domain filter step
│   └── tests/
└── document_comparison/  # Future feature
    ├── endpoint.py
    ├── handler.py
    └── tests/
```
The key principle: adding `document_comparison/` should not require modifying any file in `domain_chat/` or `document_upload/`. Dependencies between features go through shared interfaces, not direct imports.

*Registration pattern:*
Features register themselves with the FastAPI app:
```python
# main.py
from features.document_upload import register as register_upload
from features.domain_chat import register as register_domain

register_upload(app)
register_domain(app)
```
A new feature is added with one import and one registration call. No `if/elif` chains.

**What separates good from great:**
A great candidate balances clean architecture with pragmatism — don't turn a 5-file orchestrator into a 50-file framework. They propose: refactor to the pipeline pattern for the main request flow, use feature modules for genuinely separate features (upload, comparison), but keep simple additions (like domain filtering) as straightforward modifications to existing steps rather than over-abstracting. They cite YAGNI — only build the extension points for patterns that have already repeated at least twice. They mention that the existing strategy pattern (factory + base class + implementations) is the right pattern for orchestration strategies — the same pattern should apply to features. They think about documentation: a new developer should be able to look at an existing feature module as a template and copy the structure for their new feature.

---

*Prepared — May 2026*
