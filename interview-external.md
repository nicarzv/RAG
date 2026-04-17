# Interview Questions — External RAG Consultant

**Context:** Senior consultant to support GPT-RAG accelerator development. Python, Azure, Container Apps, CI/CD, AI Foundry, frontend.
**Duration:** ~25 minutes
**Format:** 7 questions, 3 priority + 4 additional

---

## Priority Questions (must ask)

### Q5 — Frontend: streaming responses

**Ask:**
Our RAG chatbot has a Chainlit-based frontend with some custom React components like the feedback form. It streams responses from the orchestrator API. Users complain that when the answer is long, nothing appears on screen for 5-6 seconds and then the entire response shows up at once. What's going wrong and how would you fix it?

**What a good answer covers:**
The frontend isn't consuming the stream properly — it's waiting for the full HTTP response to complete instead of reading chunks as they arrive. The fix is to use the `ReadableStream` API (or `EventSource` for SSE) to process partial responses as they come in and render tokens incrementally. The orchestrator is likely already streaming (Server-Sent Events or chunked transfer encoding), but the frontend `fetch` call is doing `await response.json()` which buffers everything. Should switch to reading `response.body.getReader()` and appending each chunk to the UI as it arrives. A strong candidate mentions: handling markdown rendering on partial text (incomplete code blocks mid-stream), showing a typing indicator, and the UX decision of whether to render raw tokens or wait for complete sentences.

---

### Q6 — Backend: REST API under concurrent load

**Ask:**
Our orchestrator app handles user queries — each request calls Azure AI Search, then calls the LLM directly for a streaming completion. With 200 concurrent users, how would you design this and where are the main pain points?

**What a good answer covers:**

*Baseline — what the candidate should know:*
Use an async framework (FastAPI + uvicorn) so inbound connections don't block each other — each request is a coroutine, not a thread. For outbound calls, use an async HTTP client (`aiohttp`, `httpx`) with explicit timeouts on every call so a slow OpenAI response doesn't hold a connection open forever. Use connection pooling (reuse HTTP sessions instead of creating new ones per request). For cascade prevention: if OpenAI starts timing out, you need a circuit breaker pattern — after N consecutive failures, stop sending requests for a cooldown period instead of piling up 200 requests all waiting 30 seconds each. A strong answer mentions: setting separate timeouts for search (fast, should be <1s) vs OpenAI (slower, 10-15s is normal for streaming), returning a useful error to the user early rather than letting them wait for a timeout, and that without connection pooling and timeouts you can exhaust file descriptors or memory and crash the whole process.

*What our orchestrator actually does (use to probe deeper):*
Our orchestrator is fully async (FastAPI + uvicorn, all handlers async). It uses singleton clients that are pre-warmed at startup — SearchClient (aiohttp session), AsyncAzureOpenAI, AgentsClient, CosmosDBClient all created once and reused across requests. Token caching via IdentityManager singleton. This is solid.

*What our orchestrator is missing (the interesting follow-up):*
If the candidate gives a strong answer, you can tell them: "Our app already does async with connection pooling and pre-warming. But what would you look at next?" The real gaps in our code are:

1. **No concurrency limits** — there are zero semaphores. All 200 requests fire outbound calls simultaneously with no throttling. If the Azure OpenAI deployment has a TPM cap, the first 50 requests succeed and the remaining 150 all get 429s at the same time.

2. **Minimal timeout configuration** — the aiohttp session for AI Search has no explicit request timeout (uses aiohttp defaults). Only the JWKS token fetch has an explicit 10s timeout. If AI Search becomes slow, requests hang on the default timeout (300s).

3. **429 handling only for embeddings** — the `get_embeddings()` method reads the `retry-after-ms` header and retries once. But the Search client and Agent Service calls have no 429 handling at all. Under load, Search rate limits would propagate as unhandled errors.

4. **Single uvicorn worker** — the default setup runs one uvicorn worker. Async handles concurrency for I/O-bound work, but any CPU-bound processing (JSON parsing, prompt construction) blocks the entire event loop. A strong candidate would ask about worker count.

A top-tier candidate would identify these gaps without being told — they'd ask about semaphores, timeouts, and what happens when one downstream service starts returning 429s while others are fine.

---

### Q7 — Deployment: build process & Docker dependency

**Ask:**
We deploy by logging into a Windows VM and running `azd deploy` from the app folder. If Docker isn't running, it fails. Why does `azd deploy` need Docker, and what are the alternatives?

**What a good answer covers:**
`azd deploy` builds the container image locally using Docker before pushing it to ACR — that's why Docker must be running on the machine. The Dockerfile defines the full build: base image, system dependencies, pip install, copy code. Without Docker, there's nothing to execute those build steps. Alternatives: use ACR Build Tasks (`az acr build`) which builds the image in the cloud — you send the Dockerfile and source code to ACR, it builds the image server-side, no local Docker needed. Or move the entire build to a CI pipeline (GitHub Actions, Azure DevOps) where the pipeline runner has Docker. A strong answer discusses trade-offs: ACR Build Tasks are simpler (no Docker anywhere) but slower and harder to debug build failures. A CI pipeline is the proper solution — the VM becomes unnecessary, builds are reproducible, and you get a full audit trail. The best candidates will mention that the current setup is also a security concern — whoever has access to that VM can deploy anything to production, and there's no review gate between code change and deployment.

---

## Additional Questions (use if time permits)

### Q1 — Python: async concurrency

**Ask:**
Your orchestrator app serves 50 concurrent users, each query triggers a search call to Azure AI Search and then a completion call to Azure OpenAI — both are HTTP calls that take 1-3 seconds each. How do you handle this in Python without blocking?

**What a good answer covers:**
Use `async/await` with `aiohttp` or `httpx` — each user request is a coroutine, and while one is waiting for the OpenAI response, the event loop serves other users' requests. Mention that `asyncio` is concurrency not parallelism — one thread, many coroutines switching during I/O waits. A strong answer mentions what happens under load: rate limiting from OpenAI (429s), how to handle backoff and retries, and that you might use a semaphore to cap how many completion calls run at once so you don't blow through your TPM quota.

---

### Q2 — Production debugging: silent retrieval failure

**Ask:**
We deploy a new version of the orchestrator. Users report that the chatbot answers are no longer grounded in their documents — it gives generic answers, sometimes hallucinating. The app is running, no errors in the logs, health probes pass. What could be going wrong?

**What a good answer covers:**
The model is responding but not getting search results — so the problem is between the orchestrator and AI Search, not the model itself. Could be: a search configuration changed (wrong index name, search endpoint), the OBO token for permission-filtered search is failing silently so the search returns zero results, a new code path is skipping the search step entirely, or the prompt template changed and no longer includes the retrieved context. A strong answer walks through how to isolate: check if the search call is actually happening (trace the request), check what the search is returning (zero results vs irrelevant results), check what the model is receiving in its prompt (are the chunks actually there?). The best candidates will say "the first thing I'd do is compare the actual prompt sent to the model before and after the deployment — that tells you immediately if the issue is retrieval or prompt construction."

---

### Q3 — RAG: chunking issues

**Ask:**
We ingest 10,000 SharePoint documents into a search index for a RAG chatbot. Users complain that when they ask about a specific policy, the chatbot either misses the answer entirely or gives a partial answer from the middle of a paragraph. What's likely going wrong in the ingestion pipeline, and how would you investigate?

**What a good answer covers:**
The problem is almost certainly in how documents are chunked. If chunks are too small, the relevant information might be split across two chunks and neither one contains enough context to answer. If there's no overlap between chunks, information at boundaries is lost. The candidate should say: look at the actual chunks in the search index for the document users are asking about — find it by title or parent_id, read the chunk content, and check whether the answer is actually there. If it's split across chunks, increase overlap or chunk size. If the chunker broke the document in weird places (mid-table, mid-sentence), the issue is the splitter logic — maybe the document layout wasn't parsed correctly. A strong answer also considers: maybe the document was never ingested (check the indexer logs), maybe the search isn't returning the right chunks (test the search query directly in AI Search), or maybe the chunks are there but ranked too low (top_k too small, or the query doesn't match the chunk's language).

---

### Q4 — Azure: document-level security

**Ask:**
Our RAG chatbot needs to enforce document-level permissions — users should only see answers based on documents they have access to in SharePoint. How would you approach this architecturally? What Azure services and identity mechanisms are involved?

**What a good answer covers:**
User authenticates via Entra ID (Azure AD). The chatbot gets the user's token and exchanges it for a search-scoped token using the On-Behalf-Of (OBO) flow. That token is passed to Azure AI Search, which filters results based on the user's identity and group memberships matched against permission fields stored in the index. Those permission fields are populated during ingestion by extracting ACLs from SharePoint via the Graph API. A strong answer recognizes the two sides of this: the write side (ingestion stamps permissions into the index) and the read side (query time filters by user identity). Bonus: mentions what happens when permissions change in SharePoint — there's a sync lag, and the candidate should acknowledge this as a design trade-off.

---

*Prepared — April 2026*
