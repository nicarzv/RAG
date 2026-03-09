# GPT-RAG — Custom Azure Apps & Identities

> All apps, identities, and registrations created by `azd provision` + `azd deploy`
> **Scope:** Single Agent + SharePoint (initial deployment)

---

## 1. Container Apps (Application Workloads)

GPT-RAG deploys **4 Container Apps** into a shared Container Apps Environment. Each runs as a Python service on port 80 (internal), behind HTTPS ingress. All use the `main` workload profile (D4 SKU, 0–1 instances).

### 1.1 Orchestrator

| Property | Value |
|----------|-------|
| **Service name** | `orchestrator` |
| **Canonical name** | `ORCHESTRATOR_APP` |
| **Repository** | github.com/Azure/gpt-rag-orchestrator (v2.4.1) |
| **Ingress** | External (HTTPS, TLS enforced) |
| **Replicas** | min: 1, max: 1 |
| **Resources** | 0.5 vCPU, 1 GiB RAM |
| **Dapr** | Enabled (appId: `orchestrator`, port 80, HTTP) |
| **Purpose** | Receives queries from the Frontend, loads system prompts from Cosmos DB, executes hybrid search against AI Search with user token propagation, calls OpenAI for grounded answers, saves conversation history |

**What it connects to:**

| Service | How | Why |
|---------|-----|-----|
| App Configuration | Managed identity | Discover all service endpoints at startup |
| Azure OpenAI | Managed identity | Chat completions (gpt-4o/gpt-5-mini) |
| AI Search | Managed identity + user token header | Hybrid search with permission filtering |
| Cosmos DB | Managed identity | Read prompts, read/write conversations |
| Key Vault | Managed identity | Read secrets |
| Application Insights | Connection string | Telemetry, tracing |
| Blob Storage | Managed identity | Read document blobs |

---

### 1.2 Frontend (Web UI)

| Property | Value |
|----------|-------|
| **Service name** | `frontend` |
| **Canonical name** | `FRONTEND_APP` |
| **Repository** | github.com/Azure/gpt-rag-ui (v2.2.1) |
| **Ingress** | External (HTTPS, TLS enforced) |
| **Replicas** | min: 1, max: 1 |
| **Resources** | 0.5 vCPU, 1 GiB RAM |
| **Dapr** | Enabled (appId: `frontend`, port 80, HTTP) |
| **Purpose** | User-facing chat interface. Handles Entra ID authentication, forwards queries + user tokens to the Orchestrator, displays streaming responses, collects user feedback |

**What it connects to:**

| Service | How | Why |
|---------|-----|-----|
| App Configuration | Managed identity | Discover orchestrator URL and settings |
| Orchestrator | HTTPS (internal via Dapr or direct) | Forward user queries + tokens |
| Blob Storage | Managed identity | Read/serve uploaded files, SAS token generation |
| Key Vault | Managed identity | Read secrets (e.g. API keys if enabled) |
| Application Insights | Connection string | User interaction telemetry |

**Important:** The Frontend has **no direct access** to OpenAI, AI Search, or Cosmos DB. It only talks to the Orchestrator.

---

### 1.3 Ingestion (Data Processing)

| Property | Value |
|----------|-------|
| **Service name** | `dataingest` |
| **Canonical name** | `DATA_INGEST_APP` |
| **Repository** | github.com/Azure/gpt-rag-ingestion (v2.2.2) |
| **Ingress** | External (HTTPS, TLS enforced) |
| **Replicas** | min: 1, max: 1 |
| **Resources** | 0.5 vCPU, 1 GiB RAM |
| **Dapr** | Enabled (appId: `dataingest`, port 80, HTTP) |
| **Purpose** | Connects to SharePoint via Graph API, extracts documents, chunks text, generates embeddings, pushes chunks + vectors + ACL metadata to the AI Search index |

**What it connects to:**

| Service | How | Why |
|---------|-----|-----|
| App Configuration | Managed identity | Discover all service endpoints |
| SharePoint Online | Entra app registration (client credentials) | Read documents and permissions via Graph API |
| Azure OpenAI | Managed identity | Generate embeddings (text-embedding-3-large) |
| AI Search | Managed identity | Write chunks + vectors + ACLs to the index |
| Cosmos DB | Managed identity | Read data source configuration |
| Blob Storage | Managed identity | Store extracted documents and images |
| Key Vault | Managed identity | Read secrets |

---

### 1.4 MCP Server (not used in initial scope)

| Property | Value |
|----------|-------|
| **Service name** | `mcp` |
| **Canonical name** | `MCP_APP` |
| **Repository** | github.com/Azure/gpt-rag-mcp (v0.3.5) |
| **Ingress** | External (HTTPS, TLS enforced) |
| **Replicas** | min: 1, max: 1 |
| **Resources** | 0.5 vCPU, 1 GiB RAM |
| **Dapr** | Enabled (appId: `mcp`, port 80, HTTP) |
| **Purpose** | Model Context Protocol server for external tool integration. Not used in Single Agent strategy |

**Note:** Deployed by default (`deployMcp: true`) but inactive until you switch to MCP strategy. Has the broadest RBAC (includes `StorageQueueDataContributor` for async tool processing).

---

## 2. User-Assigned Managed Identities

GPT-RAG creates **8 user-assigned managed identities (UAIs)**. Each identity is scoped to a specific resource or app, following least-privilege principles.

| # | Identity Name Pattern | Assigned To | Purpose |
|---|----------------------|-------------|---------|
| 1 | `id-ca-{token}-orchestrator` | Orchestrator Container App | Service-to-service auth for orchestrator |
| 2 | `id-ca-{token}-frontend` | Frontend Container App | Service-to-service auth for frontend |
| 3 | `id-ca-{token}-dataingest` | Ingestion Container App | Service-to-service auth for ingestion |
| 4 | `id-ca-{token}-mcp` | MCP Container App | Service-to-service auth for MCP server |
| 5 | `id-{cosmosAccountName}` | Cosmos DB account | Cosmos DB data-plane operations |
| 6 | `id-{searchServiceName}` | AI Search service | Search service operations |
| 7 | `id-{containerEnvName}` | Container Apps Environment | Environment-level operations |
| 8 | `id-{acrName}` | Container Registry | ACR image management |

**Additionally created (for infrastructure):**
| # | Identity | Purpose |
|---|----------|---------|
| 9 | `id-{aiFoundryAccountName}` | AI Foundry account operations |
| 10 | `id-{vmName}` (if VM deployed) | Jumpbox VM for admin access |

**How apps use their identity:** Each Container App receives its UAI's `clientId` as the `AZURE_CLIENT_ID` environment variable. Combined with `AZURE_TENANT_ID`, the app SDK uses `DefaultAzureCredential` to authenticate.

---

## 3. RBAC Role Assignments per App

### 3.1 Container App Roles

| RBAC Role | Orch | Front | Ingest | MCP | Purpose |
|-----------|:----:|:-----:|:------:|:---:|---------|
| AppConfigurationDataReader | ✅ | ✅ | ✅ | ✅ | Read App Configuration |
| CognitiveServicesUser | ✅ | — | ✅ | ✅ | AI Foundry services |
| CognitiveServicesOpenAIUser | ✅ | — | ✅ | ✅ | OpenAI model calls |
| CosmosDBBuiltInDataContributor | ✅ | — | ✅ | ✅ | Cosmos DB read/write |
| SearchIndexDataReader | ✅ | — | — | — | Query search index |
| SearchIndexDataContributor | — | — | ✅ | ✅ | Write to search index |
| StorageBlobDataReader | ✅ | ✅ | — | — | Read blobs |
| StorageBlobDataContributor | — | — | ✅ | ✅ | Write blobs |
| StorageBlobDelegator | — | ✅ | — | — | Generate SAS tokens |
| StorageQueueDataContributor | — | — | — | ✅ | Async queue processing |
| KeyVaultSecretsUser | ✅ | ✅ | ✅ | ✅ | Read secrets |
| AcrPull | ✅ | ✅ | ✅ | ✅ | Pull container images |

### 3.2 Deployer Principal Roles (during `azd provision`)

The user running `azd provision` receives these roles temporarily:

| Role | Purpose |
|------|---------|
| CosmosDBBuiltInDataContributor | Populate Cosmos DB containers |
| SearchServiceContributor | Manage search service |
| SearchIndexDataContributor | Create and populate indexes |
| KeyVaultContributor | Manage Key Vault |
| KeyVaultSecretsOfficer | Inject secrets |
| StorageBlobDataContributor | Upload assets |
| CognitiveServicesContributor | Configure AI services |
| CognitiveServicesOpenAIUser | Test model deployments |
| AppConfigurationDataOwner | Populate App Configuration |
| AcrPush | Push container images |
| ContainerAppsContributor | Manage Container Apps |
| ManagedIdentityOperator | Assign managed identities |

### 3.3 Infrastructure Identity Roles

| Identity | Role | Target Resource |
|----------|------|-----------------|
| AI Search UAI | StorageBlobDataReader | Storage Account |
| AI Search UAI | SearchIndexDataReader | AI Search (self) |
| AI Search UAI | SearchServiceContributor | AI Search (self) |
| Container Env UAI | StorageBlobDataReader | Storage Account |

---

## 4. Entra ID App Registration (Manual Prerequisite)

GPT-RAG requires **one Entra ID app registration** that you create **before** deployment. This is the only manual identity step.

### 4.1 Purpose

The app registration serves two functions:
1. **User authentication** — End users sign in to the Frontend via Entra ID
2. **SharePoint access** — The Ingestion component reads documents via Microsoft Graph API

### 4.2 What You Need to Create

| Property | Value |
|----------|-------|
| **Name** | e.g. `gpt-rag-{environment}` |
| **Type** | Single-tenant (your org only) |
| **Redirect URI** | `https://{frontend-app-url}/.auth/login/aad/callback` |
| **Client secret** | Yes — stored in Key Vault after provisioning |

### 4.3 API Permissions Required

| API | Permission | Type | Why |
|-----|-----------|------|-----|
| Microsoft Graph | `User.Read` | Delegated | Basic user profile for authentication |
| Microsoft Graph | `Sites.Read.All` | Application | Read SharePoint site content |
| Microsoft Graph | `Files.Read.All` | Application | Read files from SharePoint document libraries |
| Microsoft Graph | `GroupMember.Read.All` | Application | Read group memberships for ACL synchronization |

**Admin consent required:** Yes — an Azure AD admin must grant consent for the Application permissions.

### 4.4 How It's Used at Runtime

| Component | Uses App Registration For |
|-----------|--------------------------|
| Frontend | Entra ID interactive login (OAuth2 / OIDC) |
| Ingestion | Client credentials flow to read SharePoint via Graph API |
| Orchestrator | Does NOT use the app registration (uses managed identity only) |

### 4.5 Secrets Management

| Secret | Stored In | Populated By |
|--------|-----------|-------------|
| Client ID | App Configuration (`AZURE_CLIENT_ID` for auth) | postprovision script |
| Client Secret | Key Vault | Manual or postprovision script |
| Tenant ID | App Configuration + Container App env var | Bicep (automatic) |

---

## 5. Azure OpenAI Model Deployments

Two model deployments are created inside the AI Foundry account:

### 5.1 Chat Model

| Property | Value |
|----------|-------|
| **Deployment name** | `chat` |
| **Canonical name** | `CHAT_DEPLOYMENT_NAME` |
| **Model** | `gpt-5-mini` (configurable in `main.parameters.json`) |
| **Format** | OpenAI |
| **SKU** | GlobalStandard |
| **Capacity** | 40K TPM |
| **API version** | 2025-01-01-preview |
| **Used by** | Orchestrator (chat completions) |

### 5.2 Embedding Model

| Property | Value |
|----------|-------|
| **Deployment name** | `text-embedding` |
| **Canonical name** | `EMBEDDING_DEPLOYMENT_NAME` |
| **Model** | `text-embedding-3-large` |
| **Format** | OpenAI |
| **SKU** | Standard |
| **Capacity** | 40K TPM |
| **API version** | 2025-01-01-preview |
| **Used by** | Ingestion (generate embeddings at 3072 dimensions) |

---

## 6. Container Apps Environment

| Property | Value |
|----------|-------|
| **Workload profiles** | `Consumption` (serverless) + `main` (D4, 0–1 instances) |
| **Subnet** | `aca-environment-subnet` (/24, 256 IPs) |
| **Service endpoints** | CognitiveServices, AzureCosmosDB |
| **Dapr** | Enabled on all apps (service-to-service invocation) |
| **Private endpoint** | Yes (when network isolation enabled) |
| **Log destination** | Log Analytics via Application Insights |

### Container App Common Settings

All 4 Container Apps share these settings:

| Setting | Value |
|---------|-------|
| **Ingress** | External, HTTPS only, TLS enforced |
| **Target port** | 80 (internal) |
| **Transport** | Auto |
| **Allow insecure** | No |
| **Initial image** | `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest` (replaced at `azd deploy`) |
| **CPU** | 0.5 vCPU |
| **Memory** | 1.0 GiB |

### Environment Variables Injected by Bicep

Every Container App receives these env vars at creation:

| Variable | Value | Purpose |
|----------|-------|---------|
| `APP_CONFIG_ENDPOINT` | `https://{appConfigName}.azconfig.io` | Discover all other services |
| `AZURE_TENANT_ID` | Subscription tenant ID | For DefaultAzureCredential |
| `AZURE_CLIENT_ID` | Container App's UAI client ID | For DefaultAzureCredential |

All other configuration (endpoints, model names, feature flags) is read from **App Configuration** at runtime, not from environment variables.

---

## 7. Storage Containers (Blob)

| Container | Canonical Name | Purpose |
|-----------|---------------|---------|
| `documents` | `DOCUMENTS_STORAGE_CONTAINER` | Extracted SharePoint documents |
| `documents-images` | `DOCUMENTS_IMAGES_STORAGE_CONTAINER` | Extracted images from documents |
| `nl2sql` | `NL2SQL_STORAGE_CONTAINER` | NL2SQL schema files (future use) |

---

## 8. Cosmos DB Containers

| Container | Canonical Name | Purpose |
|-----------|---------------|---------|
| `conversations` | `CONVERSATIONS_DATABASE_CONTAINER` | Chat history per user session |
| `datasources` | `DATASOURCES_DATABASE_CONTAINER` | Data source connection config |
| `prompts` | `PROMPTS_CONTAINER` | System prompts (runtime-editable) |
| `mcp` | `MCP_CONTAINER` | MCP tool definitions (future use) |

---

## 9. Deployment Feature Flags

These flags in `main.parameters.json` control what gets deployed:

| Flag | Default | What It Controls |
|------|---------|-----------------|
| `deployAiFoundry` | true | AI Foundry Hub + Project + model deployments |
| `deployAppConfig` | true | App Configuration store |
| `deployAppInsights` | true | Application Insights + Log Analytics |
| `deployCosmosDb` | true | Cosmos DB account + containers |
| `deployContainerApps` | true | All 4 Container Apps |
| `deployContainerRegistry` | true | Azure Container Registry |
| `deployContainerEnv` | true | Container Apps Environment |
| `deployKeyVault` | true | Azure Key Vault |
| `deploySearchService` | true | Azure AI Search |
| `deployStorageAccount` | true | Azure Storage Account |
| `deployMcp` | true | MCP Container App |
| `deployVM` | true | Jumpbox VM |
| `deployNsgs` | true | Network Security Groups |
| `networkIsolation` | configurable | VNet + private endpoints |
| `useUAI` | configurable | User-assigned (true) vs system-assigned (false) identities |
| `useCAppAPIKey` | configurable | API key auth between Container Apps |
| `enableAgenticRetrieval` | configurable | AI Search agentic retrieval |

---

## 10. Summary: What Each Team Needs to Know

### For the Security / Identity Team

- **1 Entra app registration** needed (manual) with `Sites.Read.All`, `Files.Read.All`, `GroupMember.Read.All` (Application) + admin consent
- **8–10 managed identities** auto-created by Bicep (all user-assigned)
- **~12 RBAC roles** assigned per Container App (all least-privilege, scoped to resource group)
- **No API keys** in environment variables — all auth via DefaultAzureCredential
- **User token propagation** via `x-ms-query-source-authorization` header to AI Search
- **Fail-closed** — no user token = no search results

### For the DevOps / Infrastructure Team

- **`azd provision`** creates ~20 Azure resources via the `bicep-ptn-aiml-landing-zone` submodule
- **`azd deploy`** pushes 4 Container Apps from component repos
- **Deployer needs elevated roles** (listed in section 3.2) — typically Owner or high-privilege Contributor on the resource group
- **VNet with 7+ subnets** and 7+ private endpoints when network isolation is enabled
- **Serialized PE deployment** — expect ~15-20 min for private endpoint creation
- **Feature flags** in `main.parameters.json` control what's deployed — review before first run

### For the Development / Customization Team

- **No application code changes** needed for initial deployment
- **Prompt customization** via Cosmos DB `prompts` container (runtime, no redeploy)
- **Index schema** changes via `config/search/search.j2` (requires `azd provision`)
- **RAI filters** via `config/aifoundry/raiblocklist.json` and `raipolicies.json`
- **Model selection** via `modelDeploymentList` in `main.parameters.json`
- **Agent strategy** set to `single_agent_rag` in App Configuration (changeable at runtime)
- **Dapr enabled** on all apps — service-to-service calls can use Dapr invocation

### For the Architecture Team

- **Multi-repo** model — each component is independently versioned and deployable
- **App Configuration** is the central service registry (no hardcoded endpoints)
- **Two-layer identity** — managed identities (service-to-service) + user tokens (document security)
- **Zero-Trust** — all services behind private endpoints, all auth via RBAC, fail-closed queries
- **Horizontal scaling** — Container Apps can scale 1→N replicas (currently set to 1)
- **Dapr sidecar** — enables service discovery and invocation between Container Apps
