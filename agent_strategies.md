# GPT-RAG Orchestrator — Agent Strategies (v2.5.0)

> **Verified against:** gpt-rag-orchestrator v2.5.0, azure-search-documents 11.7.0b2, agent-framework-core 1.0.0b260116, Chainlit 2.9.4

---

## 1. Overview

The GPT-RAG orchestrator v2.5.0 ships with **6 agent strategies**, configured via the `AGENT_STRATEGY` App Configuration key. The default is `maf_lite`.

All strategies inherit from `BaseAgentStrategy` (`strategies/base_agent_strategy.py`), which provides:
- Credential management via `IdentityManager` (Entra ID / Managed Identity)
- `AIProjectClient` for Foundry interactions
- Cosmos DB connectivity via singleton `get_cosmosdb_client()`
- Prompt loading from file (`prompts/<namespace>/`) or Cosmos DB (`PROMPT_SOURCE=cosmos`)
- Model name resolution via `CHAT_DEPLOYMENT_NAME`

The strategy is selected at runtime in `Orchestrator.create()`:
```python
agentic_strategy_name = cfg.get("AGENT_STRATEGY", "maf_lite")
instance.agentic_strategy = await AgentStrategyFactory.get_strategy(agentic_strategy_name)
```

### Strategy enum (`strategies/agent_strategies.py`)

```python
class AgentStrategies(Enum):
    SINGLE_AGENT_RAG = "single_agent_rag"
    MULTIAGENT        = "multiagent"
    MCP               = "mcp"
    MAF_AGENT_SERVICE = "maf_agent_service"
    MAF_LITE          = "maf_lite"
    MULTIMODAL        = "multimodal"
    NL2SQL            = "nl2sql"
```

> **Note:** `MULTIAGENT` is in the enum but has no strategy implementation file in v2.5.0. It is reserved for future use.

---

## 2. Strategy Details

### 2.1 `maf_lite` (default)

**File:** `strategies/maf_lite_strategy.py`
**Prompt namespace:** `prompts/maf/`
**Framework:** Microsoft Agent Framework (MAF) — `ChatAgent`, `UserProfileMemory`, `SearchContextProvider`
**Model access:** Direct Azure OpenAI via `OpenAIChatClient` (no Agent Service)

**How it works:**
1. Intent classification — lightweight LLM call classifies user message as `GREETING` or `QUESTION`
2. If `QUESTION` and search provider is configured, `SearchContextProvider` runs automatically (not a tool call — injected as system message)
3. `ChatAgent` streams the response with the search context + conversation history + user profile memory
4. User profile is saved as a background task (`asyncio.create_task`)

**Key characteristics:**
- Fastest latency — one intent call + one model inference, both direct to OpenAI
- No server-side threads — conversation history managed locally in Cosmos DB `conversations` container
- `UserProfileMemory` — remembers user preferences across sessions
- Intent classifier skips search for greetings/small talk
- Semantic search enabled when `SEARCH_SEMANTIC_SEARCH_CONFIG` is set
- Hybrid search (keyword + vector) when `EMBEDDING_DEPLOYMENT_NAME` is set

**Config keys (beyond base):**

| Key | Default | Purpose |
|-----|---------|---------|
| `AI_FOUNDRY_ACCOUNT_ENDPOINT` | — (required) | Account-level endpoint for `OpenAIChatClient` |
| `OPENAI_API_VERSION` | `2025-04-01-preview` | API version for OpenAI calls |
| `SEARCH_SERVICE_QUERY_ENDPOINT` | `None` | Azure AI Search endpoint (search disabled if not set) |
| `SEARCH_RAG_INDEX_NAME` | `None` | Search index name |
| `SEARCH_RAGINDEX_TOP_K` | `3` | Number of search results |
| `SEARCH_SEMANTIC_SEARCH_CONFIG` | `None` | Semantic ranking configuration name |
| `EMBEDDING_DEPLOYMENT_NAME` | `None` | Embedding model for hybrid search (keyword-only if not set) |
| `MAX_CONTENT_CHARS` | `4000` | Max characters per document chunk sent to model |
| `CHAT_HISTORY_MAX_MESSAGES` | `10` | Recent messages included as conversation history |
| `MAX_COMPLETION_TOKENS` | `4096` | Hard cap on output tokens |
| `REASONING_EFFORT` | `medium` | Reasoning effort for supporting models (`low`/`medium`/`high`) |
| `CONVERSATIONS_DATABASE_CONTAINER` | `conversations` | Cosmos DB container for conversations and user profiles |

---

### 2.2 `maf_agent_service`

**File:** `strategies/maf_agent_service_strategy.py`
**Prompt namespace:** `prompts/maf/` (shared with `maf_lite`)
**Framework:** Microsoft Agent Framework (MAF) — `ChatAgent`, `UserProfileMemory`, `SearchContextProvider`
**Model access:** Azure AI Foundry Agent Service V2 via `AzureAIAgentClient`

**How it works:**
1. Creates or resumes a server-side thread via `AzureAIAgentClient`
2. `SearchContextProvider` runs automatically (same as `maf_lite`)
3. `ChatAgent` streams the response — model may decide tool calls (double inference)
4. `thread_id` stored in conversation document for resumption
5. User profile saved after flow

**Key characteristics:**
- Server-side threads persist on Foundry — enables thread resumption
- Model decides tool calls (double inference: first to decide tools, second for response)
- Same `SearchContextProvider` and same OBO bug as `maf_lite`
- Same `UserProfileMemory` as `maf_lite`
- Higher latency than `maf_lite` due to Agent Service round-trips
- Uses request-scoped `AzureAIAgentClient` (`async with` for cleanup)

**Additional conversation fields:** `thread_id` (Foundry server-side thread ID)

**Config keys:** Same as `maf_lite` minus `AI_FOUNDRY_ACCOUNT_ENDPOINT`, `MAX_CONTENT_CHARS`, `CHAT_HISTORY_MAX_MESSAGES` (history managed server-side). Additionally requires `AI_FOUNDRY_PROJECT_ENDPOINT` (from base).

---

### 2.3 `single_agent_rag`

**File:** `strategies/single_agent_rag_strategy_v2.py`
**Prompt namespace:** `prompts/single_agent_rag/`
**Framework:** Azure AI Agents SDK (`AgentsClient`) — NOT MAF
**Model access:** Azure AI Foundry Agent Service via `AgentsClient`

**How it works:**
1. Pre-warms `AgentsClient` + agent at startup (`prewarm_agents_client()`) for zero cold-start
2. Checks if search index is empty (cached with TTL)
3. **Empty index bypass:** Routes directly to `GenAIModelClient` for lightweight streaming (skips Agent SDK entirely)
4. **Non-empty index:** Creates server-side thread, sends message, streams via Agent SDK event handlers
5. Search is a `FunctionTool` wrapping `connectors/search.py` — model decides when to call `search_knowledge_base`
6. Supports Bing grounding as additional tool

**Key characteristics:**
- Legacy V2 strategy carried over from pre-2.5.0
- Uses `connectors/search.py` with raw `aiohttp` HTTP calls (no OBO header bug)
- Search is model-initiated (function tool), not automatic
- Supports `BING_RETRIEVAL_ENABLED` + `BING_CONNECTION_ID`
- Has `SEARCH_RETRIEVAL_ENABLED` on/off flag
- Has `ALLOW_ANONYMOUS` strict mode for OBO
- Uses `SEARCH_APPROACH` config (`hybrid`/`vector`/`term`)
- Embeddings via `GenAIModelClient` (requires `EMBEDDING_DEPLOYMENT_NAME`)
- Pre-warmed singleton agent reused across requests
- Supports `AGENT_ID` to reuse a Foundry-configured agent
- No `UserProfileMemory`
- No intent classification

**Additional config keys (beyond base):**

| Key | Default | Purpose |
|-----|---------|---------|
| `AGENT_ID` | `""` | Pre-existing Foundry agent ID to reuse |
| `SEARCH_RETRIEVAL_ENABLED` | `true` | Enable/disable search tool |
| `BING_RETRIEVAL_ENABLED` | `false` | Enable Bing grounding tool |
| `BING_CONNECTION_ID` | `""` | Foundry Bing connection ID |
| `SEARCH_APPROACH` | `hybrid` | Search mode: `hybrid`, `vector`, `term` |
| `SEARCH_USE_SEMANTIC` | `false` | Enable semantic ranking |
| `ALLOW_ANONYMOUS` | `true` | Allow search without OBO token (strict mode if `false`) |

---

### 2.4 `mcp`

**File:** `strategies/mcp_strategy.py`
**Prompt namespace:** N/A (uses Semantic Kernel agent instructions)
**Framework:** Semantic Kernel — `ChatCompletionAgent`, `MCPSsePlugin`
**Model access:** Azure OpenAI via Semantic Kernel's `AzureChatCompletion`

**How it works:**
1. Connects to an external MCP server (`gpt-rag-mcp` Container App) via SSE transport
2. MCP server exposes tools the model can call
3. Uses Semantic Kernel `ChatCompletionAgent` for orchestration
4. Passes `user_context` as custom HTTP header via `h11` monkey-patch

**Key characteristics:**
- Designed for external tool servers (databases, APIs, custom retrieval)
- MCP server is a separate Container App (`gpt-rag-mcp`)
- Model decides which MCP tools to call
- User context propagated via custom HTTP header for per-user tool behavior
- No built-in search — retrieval is handled by MCP server tools

**Additional config keys:**

| Key | Default | Purpose |
|-----|---------|---------|
| `MCP_APP_ENDPOINT` | `http://localhost:80` | MCP server URL |
| `MCP_CLIENT_TIMEOUT` | `600` | MCP client timeout (seconds) |
| `MCP_APP_APIKEY` | `None` | MCP server API key |
| `MCP_SERVER_TRANSPORT` | `sse` | MCP transport type |
| `AGENT_ID` | `""` | Pre-existing Foundry agent ID |

---

### 2.5 `nl2sql`

**File:** `strategies/nl2sql_strategy.py`
**Prompt namespace:** `prompts/nl2sql/`
**Framework:** Semantic Kernel — `AzureAIAgent`, `AgentGroupChat`
**Model access:** Azure AI Foundry Agent Service via Semantic Kernel

**How it works:**
1. Creates 3 Foundry agents in parallel: Triage Agent, SQL Query Agent, Synthesizer Agent
2. `AgentGroupChat` orchestrates multi-agent conversation with termination strategy
3. Triage agent routes the question, SQL agent generates and executes queries, Synthesizer formats results
4. `NL2SQLPlugin` provides SQL execution capabilities

**Key characteristics:**
- Multi-agent pattern (3 agents per request)
- Agents created per-request (no pre-warming)
- Requires configured SQL data source via `NL2SQLPlugin`
- `ApprovalTerminationStrategy` — terminates when agent emits "TERMINATE"
- Higher latency due to multi-agent round-trips + agent creation

---

### 2.6 `multimodal`

**File:** `strategies/multimodal_strategy.py`
**Prompt namespace:** `prompts/maf/` (shared with `maf_lite`)
**Framework:** Microsoft Agent Framework (MAF) — `ChatAgent`, `UserProfileMemory`, `MultimodalSearchContextProvider`
**Model access:** Direct Azure OpenAI via `MultimodalChatClient`

**How it works:**
1. Same flow as `maf_lite` (intent classification → search → stream)
2. `MultimodalSearchContextProvider` retrieves text chunks AND related images from search index
3. Images downloaded from Azure Blob Storage
4. Image classifier filters irrelevant images (logos, decorative art, chapter dividers)
5. Text + images sent to vision-capable model (e.g., GPT-4o)

**Key characteristics:**
- Extended `maf_lite` with vision capabilities
- Dual vector search (`contentVector` + `captionVector`)
- Image classification step to filter noise
- `MultimodalChatClient` handles mixed text+image payloads
- Same `UserProfileMemory` and intent classification as `maf_lite`
- Requires vision-capable model deployment

**Additional config keys (beyond maf_lite):**

| Key | Default | Purpose |
|-----|---------|---------|
| `MULTIMODAL_MAX_IMAGES` | `10` | Max total images per request |
| `MULTIMODAL_MAX_IMAGES_PER_DOC` | `5` | Max images per document |
| `MULTIMODAL_MAX_CONTENT_CHARS` | `4000` | Max text chars per chunk |
| `MULTIMODAL_CLASSIFY_IMAGES` | `true` | Enable image relevance classification |
| `MULTIMODAL_IMAGE_CLASSIFICATION_TIMEOUT_SECONDS` | `15` | Image classification timeout |
| `MULTIMODAL_IMAGE_CLASSIFICATION_CONCURRENCY` | `2` | Parallel image classifications |
| `MULTIMODAL_VALIDATE_RESPONSE_IMAGES` | `true` | Validate images referenced in response |
| `MULTIMODAL_IMAGE_VALIDATION_TIMEOUT_SECONDS` | `15` | Image validation timeout |

---

## 3. Feature Comparison Matrix

| Feature | `maf_lite` | `maf_agent_service` | `single_agent_rag` | `mcp` | `nl2sql` | `multimodal` |
|---------|-----------|-------------------|-------------------|-------|---------|-------------|
| **Model access** | Direct OpenAI | Agent Service V2 | Agent Service (AgentsClient) | Semantic Kernel | Semantic Kernel | Direct OpenAI |
| **Search mechanism** | Automatic (ContextProvider) | Automatic (ContextProvider) | Model-initiated (FunctionTool) | Via MCP tools | N/A | Automatic (ContextProvider) |
| **OBO / ACL trimming** | Yes (needs header fix) | Yes (needs header fix) | Yes (works) | N/A | N/A | Yes (needs header fix) |
| **Server-side threads** | No | Yes | Yes | No | Yes (per-request) | No |
| **User profile memory** | Yes | Yes | No | No | No | Yes |
| **Intent classification** | Yes | No | No | No | No | Yes |
| **Bing grounding** | No | No | Yes | No | No | No |
| **Semantic search** | Yes (if configured) | Yes (if configured) | Via `SEARCH_USE_SEMANTIC` | N/A | N/A | Yes (if configured) |
| **Hybrid search** | Via `EMBEDDING_DEPLOYMENT_NAME` | Via `EMBEDDING_DEPLOYMENT_NAME` | Via `SEARCH_APPROACH` | N/A | N/A | Via `EMBEDDING_DEPLOYMENT_NAME` |
| **Cosmos DB prompts** | Yes (`PROMPT_SOURCE=cosmos`) | Yes | Yes | No | Yes | Yes |
| **Image/vision support** | No | No | No | No | No | Yes |
| **Latency** | Lowest | Medium | Medium-High | Depends on MCP | High | Low-Medium |
| **`ALLOW_ANONYMOUS` strict mode** | No (always graceful) | No (always graceful) | Yes | N/A | N/A | No (always graceful) |

---

## 4. Shared Base Configuration (all strategies)

These keys are read by `BaseAgentStrategy.__init__()` and apply to all strategies:

| Key | Default | Required | Purpose |
|-----|---------|----------|---------|
| `AI_FOUNDRY_PROJECT_ENDPOINT` | — | **Yes** | Foundry project endpoint |
| `CHAT_DEPLOYMENT_NAME` | — | **Yes** | Model deployment name (e.g., `gpt-4.1-mini`) |
| `OPENAI_API_VERSION` | `2025-04-01-preview` | No | API version |
| `PROMPT_SOURCE` | `file` | No | Prompt source: `file` or `cosmos` |
| `AGENT_STRATEGY` | `maf_lite` | No | Which strategy to load |
| `CONVERSATIONS_DATABASE_CONTAINER` | `conversations` | No | Cosmos DB container name |

---

## 5. Fix maf_lite — OBO Header Bug

### Problem

After upgrading to orchestrator v2.5.0 and switching to `maf_lite` (or `maf_agent_service`), search fails with:

```
Search failed: Invalid header: 'x-ms-query-source-authorization'
```

This affects **all users with Entra ID authentication enabled** where OBO (On-Behalf-Of) document-level security trimming is active.

### Root cause

The `SearchContextProvider` (`strategies/search_context_provider.py`) passes the OBO token as a keyword argument to the Azure SDK's `SearchClient.search()` method:

```python
# BUG: SDK does not recognize this as a keyword argument
search_params["x_ms_query_source_authorization"] = f"Bearer {obo_token}"
# ...
results = await client.search(**search_params)
```

The Azure SDK's `SearchClient.search()` does not translate `x_ms_query_source_authorization` into the HTTP header `x-ms-query-source-authorization`. It either rejects it or passes it incorrectly.

The previous `single_agent_rag` strategy worked because it used `connectors/search.py` which sends the header via raw `aiohttp` HTTP calls:

```python
# OLD (connectors/search.py) — works correctly
headers["x-ms-query-source-authorization"] = f"Bearer {search_user_token}"
```

### Fix

**File:** `src/strategies/search_context_provider.py`
**Lines:** 126–137

**Before (broken):**
```python
            if obo_token:
                search_params["x_ms_query_source_authorization"] = f"Bearer {obo_token}"
                logger.info("[SearchContextProvider] Using x-ms-query-source-authorization (OBO)")
            else:
                logger.info("[SearchContextProvider] Not sending x-ms-query-source-authorization")

            async with SearchClient(
                endpoint=self._endpoint,
                index_name=self._index_name,
                credential=self._credential,
            ) as client:
                results = await client.search(**search_params)
```

**After (fixed):**
```python
            extra_headers = {}
            if obo_token:
                extra_headers["x-ms-query-source-authorization"] = f"Bearer {obo_token}"
                logger.info("[SearchContextProvider] Using x-ms-query-source-authorization (OBO)")
            else:
                logger.info("[SearchContextProvider] Not sending x-ms-query-source-authorization")

            async with SearchClient(
                endpoint=self._endpoint,
                index_name=self._index_name,
                credential=self._credential,
            ) as client:
                results = await client.search(**search_params, headers=extra_headers)
```

### Why this works

The Azure SDK for Python forwards the `headers` dict parameter to its underlying HTTP pipeline, which sets them as actual HTTP request headers — exactly what the old `aiohttp` approach did. The OBO token exchange itself is unchanged; only the delivery mechanism to the Search service is fixed.

### Impact

- **Affected strategies:** `maf_lite`, `maf_agent_service`, `multimodal` (all three use `SearchContextProvider`)
- **Not affected:** `single_agent_rag` (uses `connectors/search.py` with raw HTTP), `mcp`, `nl2sql`
- **Fix scope:** Single file, single line change
- **After fix:** Document-level ACL trimming works identically to the old `single_agent_rag` behavior

### Verification

After applying the fix, check the orchestrator logs for:

```
[SearchContextProvider] Using x-ms-query-source-authorization (OBO)
[SearchContextProvider] Search returned N documents in X.XXs
```

If OBO is not configured (no `OAUTH_AZURE_AD_CLIENT_SECRET`), you should see:

```
[SearchContextProvider] Not sending x-ms-query-source-authorization
```

### Optional: Add ALLOW_ANONYMOUS strict mode to maf_lite

The `single_agent_rag` strategy has an `ALLOW_ANONYMOUS` config flag that blocks search entirely when OBO fails. `maf_lite` always degrades gracefully (searches without ACL trimming). To add strict mode, modify `_create_search_provider` in `maf_lite_strategy.py`:

```python
# In _create_search_provider, change the _get_obo_token closure:
allow_anonymous = cfg.get("ALLOW_ANONYMOUS", default=True, type=bool)

async def _get_obo_token() -> str | None:
    token = getattr(self, "request_access_token", None)
    return await acquire_obo_search_token(token, allow_anonymous=allow_anonymous) if token else None
```

---

## 6. Diagnostic Log Messages

When troubleshooting `maf_lite` search issues, look for these log lines in the orchestrator Container App:

### Search provider creation
- `[MafLiteStrategy] SearchContextProvider created (index=..., top_k=...)` — search configured correctly
- `[MafLiteStrategy] No search endpoint configured, skipping search` — `SEARCH_SERVICE_QUERY_ENDPOINT` missing
- `[MafLiteStrategy] No search index name configured, skipping search` — `SEARCH_RAG_INDEX_NAME` missing
- `[MafLiteStrategy] Failed to create search provider: ...` — exception during provider creation
- `[MafLiteStrategy] search_provider_init: ...s (hybrid=True)` — vector search active
- `[MafLiteStrategy] search_provider_init: ...s (hybrid=False)` — keyword only, no `EMBEDDING_DEPLOYMENT_NAME`

### Intent classification
- `[MafLiteStrategy] intent=question` — search will run
- `[MafLiteStrategy] Greeting detected — skipping search` — search skipped
- `[MafLiteStrategy] Intent classification failed: ... — defaulting to question` — classification errored, fell back to search

### Search execution
- `[SearchContextProvider] Query: "..." (top_k=3, hybrid=True/False)` — search attempted
- `[SearchContextProvider] Using x-ms-query-source-authorization (OBO)` — OBO token sent
- `[SearchContextProvider] Not sending x-ms-query-source-authorization` — no OBO token
- `[SearchContextProvider] Search returned N documents in X.XXs` — results returned
- `[SearchContextProvider] Search failed: ...` — search threw an error

### OBO token
- `[OBO] Acquired Search delegated token` — OBO exchange succeeded
- `[OBO] Missing Entra config for OBO (tenant=... client=... secret=...)` — OBO config incomplete

---

## 7. Migration Notes

### From `single_agent_rag` to `maf_lite`

1. **Apply the OBO header fix** (Section 5) if using Entra ID with document-level security
2. **Verify these App Config keys exist** (used by `maf_lite` but read differently):
   - `SEARCH_SERVICE_QUERY_ENDPOINT` — same key, but `maf_lite` uses `allow_none=True` (silent failure vs crash)
   - `SEARCH_RAG_INDEX_NAME` — same key, same note
   - `EMBEDDING_DEPLOYMENT_NAME` — optional in `maf_lite` (falls back to keyword), required by `GenAIModelClient` in `single_agent_rag`
3. **Prompt namespace change:** `prompts/single_agent_rag/main.jinja2` → `prompts/maf/main.txt` (or `.jinja2`)
4. **`SEARCH_APPROACH` is ignored** by `maf_lite`. Hybrid search is enabled solely by having `EMBEDDING_DEPLOYMENT_NAME` set.
5. **`SEARCH_USE_SEMANTIC` is ignored.** Use `SEARCH_SEMANTIC_SEARCH_CONFIG` instead.
6. **`BING_RETRIEVAL_ENABLED` is not supported** in `maf_lite`.
7. **`ALLOW_ANONYMOUS` is not supported** by default (see optional fix in Section 5).
8. **`AGENT_ID` is not used** — `maf_lite` doesn't create Foundry agents.

### From old `maf` (pre-v2.5.0) to `maf_agent_service`

The v2.5.0 release renamed `maf` to `maf_agent_service`. Update your `AGENT_STRATEGY` value in App Configuration from `maf` to `maf_agent_service`. The strategy code is otherwise compatible.

---

## 8. Architecture Decision: When to Use Which Strategy

| Scenario | Recommended Strategy | Reason |
|----------|---------------------|--------|
| Standard RAG chatbot, lowest latency | `maf_lite` | Direct OpenAI, no Agent Service overhead |
| Need server-side thread persistence | `maf_agent_service` | Foundry manages threads |
| Need Bing grounding alongside search | `single_agent_rag` | Only strategy with Bing support |
| External tool servers (APIs, databases) | `mcp` | MCP protocol for tool federation |
| Natural language to SQL queries | `nl2sql` | Multi-agent SQL pipeline |
| Documents with diagrams/images | `multimodal` | Vision model + image retrieval |
| Per-department separation (future) | Extended `maf_lite` or `single_agent_rag` | See Section 9 |

---

## 9. Known Issue: `reasoning_effort` Breaks `maf_agent_service`

### Problem

When using `maf_agent_service` with a reasoning-capable model (e.g., `gpt-5-mini`, `o3-mini`), the UI shows:

```
unexpected keyword argument 'reasoning_effort'
```

### Root cause

`maf_agent_service_strategy.py` unconditionally passes `reasoning_effort` in the options dict (line 265):

```python
options={"max_completion_tokens": self.max_completion_tokens, "reasoning_effort": self.reasoning_effort},
```

The `AzureAIAgentClient` routes through the Agent Service V2 runs API, which does **not** support `reasoning_effort` as a run parameter yet. The `agent-framework-azure-ai` SDK (1.0.0b260116) passes it through and the API rejects it.

**Why `maf_lite` works:** It uses `OpenAIChatClient` → direct Azure OpenAI API, which supports `reasoning_effort` natively.

**Why `single_agent_rag` doesn't break:** It never passes `reasoning_effort` to the Agent Service. In its direct LLM bypass path, it wraps the call in `try/except BadRequestError` and retries without it:

```python
# single_agent_rag_strategy_v2.py lines 269-273
try:
    response_stream = await self.llm_client.openai_client.chat.completions.create(**_kwargs)
except BadRequestError:
    _kwargs.pop("reasoning_effort", None)
    response_stream = await self.llm_client.openai_client.chat.completions.create(**_kwargs)
```

### Fix

**File:** `src/strategies/maf_agent_service_strategy.py`
**Line:** 265

**Before (broken):**
```python
options={"max_completion_tokens": self.max_completion_tokens, "reasoning_effort": self.reasoning_effort},
```

**After (fixed):**
```python
options={"max_completion_tokens": self.max_completion_tokens},
```

### No config-only workaround

The `REASONING_EFFORT` config key defaults to `"medium"` and is always included in the options dict. There is no way to exclude it through configuration alone. A code change is required.

### Impact

- **Affected:** `maf_agent_service` only
- **Not affected:** `maf_lite` (works natively), `single_agent_rag` (defensive coding), `multimodal` (direct OpenAI)
- Once Microsoft adds `reasoning_effort` support to the Agent Service runs API, this line can be restored

---

## 10. Multi-Department Agent Routing Architecture

### 10.1 Goal

Route users to department-specific agents, each with its own model deployment, system prompt, retrieval behavior, and tool configuration. Department identity comes from Entra ID claims (e.g., `department` attribute in the user's token).

### 10.2 Approach A: `single_agent_rag` with Per-Department `AGENT_ID`

This is the simplest path — requires minimal code changes because `single_agent_rag` already supports `AGENT_ID` for reusing pre-configured Foundry agents.

**Architecture:**

```
User (Entra ID: department=Legal)
  → Frontend passes user_context with department claim
    → Orchestrator reads department from user_context
      → Routes to AGENT_ID for Legal
        → Pre-configured Foundry agent with:
           - Model: gpt-4.1 (high accuracy for legal)
           - Instructions: legal-specific system prompt
           - Tools: search_knowledge_base (legal index)
```

**Step 1: Create department agents in Foundry portal**

Create a separate agent for each department in the Azure AI Foundry portal. Each agent gets its own:
- Model deployment (e.g., Legal → `gpt-4.1`, HR → `gpt-4.1-mini`)
- System instructions (department-specific prompts)
- Tool configuration (can point to different search indexes or tools)

Record each agent's ID from the Foundry portal.

**Step 2: Add department-to-agent mapping in App Configuration**

```
AGENT_ID_LEGAL=asst_abc123
AGENT_ID_HR=asst_def456
AGENT_ID_FINANCE=asst_ghi789
AGENT_ID_DEFAULT=asst_xyz000
```

**Step 3: Modify `single_agent_rag_strategy_v2.py` to route by department**

In `_stream_agent()`, replace the static `AGENT_ID` lookup with department-based routing:

```python
# In _stream_agent(), replace the agent resolution logic:
department = (self.user_context or {}).get("department", "").lower()
agent_id_key = f"AGENT_ID_{department.upper()}" if department else "AGENT_ID"
resolved_agent_id = self.cfg.get(agent_id_key, "") or self.cfg.get("AGENT_ID", "") or None

# Then use resolved_agent_id instead of self.existing_agent_id
```

**Step 4: Pass department from the frontend**

The `user_context` dict is already propagated from the frontend → orchestrator → strategy. Ensure the frontend extracts the `department` claim from the Entra ID token and includes it in the orchestrator request.

**Advantages:**
- Minimal code changes (routing logic only)
- Agents managed in Foundry portal (no code deployments to change prompts)
- OBO/ACL trimming works per department (each agent can use different search indexes)
- `reasoning_effort` handled defensively (no breakage)
- Pre-warming works with default agent; department agents fetched on first use and cached

**Limitations:**
- Agents are managed in Foundry portal (not in code/IaC)
- No `UserProfileMemory` across departments
- Agent creation/management is a manual portal task unless scripted via API

---

### 10.3 Approach B: `maf_agent_service` with Per-Department Configuration

This uses MAF's composable architecture. Instead of pre-configured Foundry agents, you customize the `ChatAgent` per request with department-specific settings.

**Architecture:**

```
User (Entra ID: department=Legal)
  → Frontend passes user_context with department claim
    → Orchestrator reads department from user_context
      → maf_agent_service creates ChatAgent with:
         - Model: resolved from CHAT_DEPLOYMENT_NAME_LEGAL
         - Instructions: loaded from prompts/maf/legal/main.txt
         - SearchContextProvider: pointed at legal search index
         - UserProfileMemory: department-scoped
```

**Step 1: Add department-specific config keys in App Configuration**

```
# Model per department
CHAT_DEPLOYMENT_NAME_LEGAL=gpt-4.1
CHAT_DEPLOYMENT_NAME_HR=gpt-4.1-mini
CHAT_DEPLOYMENT_NAME_FINANCE=gpt-4.1-mini

# Search index per department (optional — use same index with ACL trimming, or separate indexes)
SEARCH_RAG_INDEX_NAME_LEGAL=ragindex-legal
SEARCH_RAG_INDEX_NAME_HR=ragindex-hr

# Fallbacks
CHAT_DEPLOYMENT_NAME=gpt-4.1-mini
SEARCH_RAG_INDEX_NAME=ragindex
```

**Step 2: Create department-specific prompt files**

```
prompts/
  maf/
    main.txt              ← default fallback
    legal/
      main.txt            ← legal-specific instructions
    hr/
      main.txt            ← HR-specific instructions
    finance/
      main.txt            ← finance-specific instructions
```

Or if using Cosmos DB prompts (`PROMPT_SOURCE=cosmos`), store department-specific prompts with keys like `maf/legal/main`, `maf/hr/main`.

**Step 3: Modify `maf_agent_service_strategy.py` to resolve config per department**

```python
# In __init__ or at the start of initiate_agent_flow:
department = (self.user_context or {}).get("department", "").lower()

def _dept_config(key, default=None):
    """Try department-specific key first, fall back to base key."""
    if department:
        dept_val = cfg.get_value(f"{key}_{department.upper()}", allow_none=True)
        if dept_val:
            return dept_val
    return cfg.get(key, default)

self.model_name = _dept_config("CHAT_DEPLOYMENT_NAME")
self.search_index_name = _dept_config("SEARCH_RAG_INDEX_NAME")
```

**Step 4: Override prompt namespace by department**

```python
def _prompt_namespace(self) -> str:
    department = (self.user_context or {}).get("department", "").lower()
    if department:
        dept_path = Path(__file__).resolve().parent.parent / "prompts" / "maf" / department
        if dept_path.exists():
            return f"maf/{department}"
    return "maf"
```

**Step 5: Apply the OBO header fix (Section 5) and reasoning_effort fix (Section 9)**

Both bugs must be fixed before `maf_agent_service` is production-ready.

**Advantages:**
- `UserProfileMemory` per user (department-scoped if needed)
- Composable architecture — add new ContextProviders per department
- No Foundry portal agent management (everything in code + config)
- Prompt management via files or Cosmos DB
- Automatic search via ContextProvider (more predictable than tool-based)

**Limitations:**
- More code changes than Approach A
- Two bugs to fix first (OBO header + reasoning_effort)
- `reasoning_effort` not available until Agent Service adds support
- No pre-warming — `AzureAIAgentClient` is request-scoped
- Transient agents (no `AGENT_ID` reuse)

---

### 10.4 Recommendation

| Criteria | Approach A (`single_agent_rag`) | Approach B (`maf_agent_service`) |
|----------|-------------------------------|--------------------------------|
| **Code changes needed** | Minimal (routing only) | Moderate (routing + fixes) |
| **Bugs to fix first** | None | 2 (OBO header + reasoning_effort) |
| **Agent management** | Foundry portal | Code + App Config |
| **User profile memory** | No | Yes |
| **reasoning_effort support** | Yes (defensive) | No (until API support) |
| **Search control** | Model-initiated (smarter) | Automatic (predictable) |
| **Latency** | Medium-High | Medium |
| **Future extensibility** | Limited by Foundry agent features | High (MAF composable) |

**For immediate deployment:** Approach A (`single_agent_rag`) — works today, no bugs to fix, minimal code changes.

**For long-term architecture:** Approach B (`maf_agent_service`) — more extensible, but requires fixing two bugs first and waiting for Agent Service `reasoning_effort` support.

**Hybrid approach:** Start with Approach A for production, develop Approach B in parallel, switch when `maf_agent_service` is stabilised.
