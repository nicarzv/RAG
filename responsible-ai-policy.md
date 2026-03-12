# GPT-RAG Responsible AI (RAI) Policy

> How the GPT-RAG Solution Accelerator implements Responsible AI: configuration files, automation script, content safety categories, system prompt guardrails, monitoring, and evaluation. Everything is applied automatically during `azd provision` via the postprovision hook.

---

## 1. Architecture Overview

GPT-RAG implements RAI as **defense-in-depth** with four layers:

| Layer | What It Does | Where It's Configured |
|-------|-------------|----------------------|
| **Custom Blocklists** | Hard-blocks specific terms/patterns before they reach the model | `config/aifoundry/raiblocklist.json` |
| **Content Filtering Policies** | Severity-based filtering across four harm categories | `config/aifoundry/raipolicies.json` |
| **System Prompt Guardrails** | Domain-specific behavioral constraints embedded in the agent prompt | `prompts/{strategy}/main.txt` or Cosmos DB |
| **Document-Level Security** | Permission filters ensuring users only see authorized content | ACL fields in AI Search index |

All four layers operate independently — a request must pass through all of them to produce a response.

### Request Flow Through RAI Layers

```
User Query
    │
    ▼
┌─────────────────────────────┐
│  Layer 1: Custom Blocklists │  ← Hard block: exact term/pattern match
│  (Azure OpenAI deployment)  │     If matched → request rejected immediately
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Layer 2: Content Filters   │  ← Soft block: severity-based classification
│  (Azure OpenAI deployment)  │     Hate, Sexual, Violence, Self-Harm
└─────────────┬───────────────┘     + Jailbreak, Indirect Attack, Protected Material
              │
              ▼
┌─────────────────────────────┐
│  Layer 3: System Prompt     │  ← Behavioral constraints
│  (Orchestrator agent)       │     Domain guardrails, persona, citation rules
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Layer 4: Document Security │  ← Data access control
│  (AI Search + OBO token)    │     Permission-filtered search results
└─────────────┬───────────────┘
              │
              ▼
          Response
```

**Important:** Layers 1 and 2 are applied at the **Azure OpenAI deployment level** — they intercept both input (user prompt) and output (model response). They apply to chat model deployments only, **not** to the embedding model.

---

## 2. Configuration Files

### 2.1 Custom Blocklists — `config/aifoundry/raiblocklist.json`

**Purpose:** Define organization-specific terms and patterns that should be hard-blocked. When a user's input or the model's output matches a blocklist term, the request is rejected immediately — the model never sees it.

**Behavior:**
- Terms are matched as exact strings or patterns
- Applied as **hard-block rules** — no severity threshold, just match/reject
- Multiple blocklists can be defined (e.g., `competitor-names`, `confidential-codes`)
- Each blocklist can contain multiple items

**Use cases:**
- Block competitor product names from appearing in responses
- Prevent confidential internal codes or project names from being generated
- Block profanity or slurs specific to your organization's context
- Prevent the model from referencing specific individuals

### 2.2 Content Filtering Policies — `config/aifoundry/raipolicies.json`

**Purpose:** Define severity thresholds for four harm categories. Content is classified by severity and blocked when it exceeds the configured threshold.

**Behavior:**
- Applied as **soft-block rules** — configurable per deployment
- Each category has an independent severity threshold
- Severity levels: **low**, **medium**, **high**
- The threshold defines the minimum severity at which content is blocked

**Categories:**

| Category | What It Detects | Default Threshold |
|----------|----------------|-------------------|
| **Hate** | Content promoting hatred or discrimination based on protected characteristics (race, gender, religion, etc.) | medium |
| **Sexual** | Sexually explicit content | medium |
| **Violence** | Content depicting or promoting violence | medium |
| **Self-Harm** | Content related to self-harm or suicide | medium |

**Additional filters** (beyond the four core categories, configurable via REST API):

| Filter | Behavior |
|--------|----------|
| **Jailbreak** | Detects prompt injection / jailbreak attempts — binary block, no severity levels |
| **Indirect Attack** | Detects indirect prompt injection via retrieved content — binary block |
| **Protected Material** | Detects unauthorized use of copyrighted content — binary block |

**REST API example** (full policy with all filters):

```json
{
  "name": "enterprise-safety-policy",
  "mode": "blocking",
  "filters": {
    "hate": {"severity": "medium", "action": "block"},
    "violence": {"severity": "medium", "action": "block"},
    "selfHarm": {"severity": "medium", "action": "block"},
    "sexual": {"severity": "medium", "action": "block"},
    "jailbreak": {"action": "block"},
    "indirectAttack": {"action": "block"},
    "protectedMaterial": {"action": "block"}
  },
  "customBlocklists": ["competitor-names", "confidential-codes"]
}
```

---

## 3. Automation Script — `config/aifoundry/setup.py`

The postprovision hook (`azd provision`) runs `setup.py` to apply RAI configuration automatically. The script performs four operations in sequence:

| Step | Operation | SDK Used |
|------|-----------|----------|
| 1 | **Create custom blocklists** — reads `raiblocklist.json`, creates blocklist items | `CognitiveServicesManagementClient` |
| 2 | **Create content filtering policies** — reads `raipolicies.json`, normalizes severity levels, creates filter policies | `CognitiveServicesManagementClient` |
| 3 | **Associate policies with deployments** — links content filter policies to Azure OpenAI model deployments | `CognitiveServicesManagementClient` |
| 4 | **Inject secrets** — stores evaluation API keys and configuration in Key Vault | Key Vault SDK |

**Critical details:**
- Step 3 applies policies to **chat model deployments only** — the embedding model (`text-embedding-3-large`) is excluded because it doesn't generate user-facing content
- The script uses `CognitiveServicesManagementClient` (management plane), not the data plane SDK — this means the deployer identity needs `CognitiveServicesContributor` role
- The script is idempotent — re-running `azd provision` reapplies policies without duplicating them

### RBAC Requirement

| Role | Scope | Who Needs It |
|------|-------|-------------|
| `CognitiveServicesContributor` | Cognitive Services account | The deployer identity running `setup.py` |

---

## 4. System Prompt Guardrails (Layer 3)

The system prompt is the third layer of RAI defense. It defines behavioral constraints for the AI agent — what it should and shouldn't do, how it should cite sources, and domain-specific guardrails.

### Prompt Location

| PROMPT_SOURCE | Location | Behavior |
|---------------|----------|----------|
| `file` (default) | `prompts/{strategy_type}/{name}.txt` or `.jinja2` | Read from disk at container startup; changes require redeployment |
| `cosmos` | Cosmos DB `prompts` container, document ID: `{strategy}_{prompt_name}` | Read at runtime per request; changes take effect immediately |

### Prompt Files by Strategy

```
prompts/
├── single_agent_rag/
│   ├── main.txt           # Primary system prompt for default strategy
│   └── *.jinja2           # Dynamic prompts with variables
├── mcp/
│   └── main.txt           # MCP strategy prompt
└── nl2sql/
    ├── triage_agent.txt       # Routes questions to SQL or docs
    ├── sqlquery_agent.txt     # Generates and executes SQL
    └── syntetizer_agent.txt   # Combines results into final answer
```

### What to Include in System Prompt Guardrails

- **Persona and tone** — how the agent should present itself
- **Citation rules** — how to reference source documents
- **Domain boundaries** — what topics the agent should and shouldn't answer
- **Refusal behavior** — how to decline out-of-scope questions
- **Output format** — response structure expectations

### Runtime Editing (Recommended for Production)

Set `PROMPT_SOURCE=cosmos` in App Configuration to enable runtime prompt editing without redeployment:
1. Run `upload_prompts.py` to seed the `prompts` container initially
2. Edit prompts directly in Cosmos DB Data Explorer or via API
3. Changes take effect on the next request — no container restart needed

---

## 5. How to Customize RAI

### 5.1 Adjusting Content Filtering Thresholds

1. Edit `config/aifoundry/raipolicies.json` — change severity levels (low/medium/high)
2. Run `azd provision` — the postprovision hook reapplies policies
3. Verify via Azure CLI:

```bash
az cognitiveservices account list-rai-policies \
  --name {cognitive-services-account-name} \
  --resource-group {rg-name}
```

**Guidance on thresholds:**
- **Low** — strictest: blocks even mildly suggestive content (good for children-facing or highly regulated applications)
- **Medium** — balanced: blocks moderately harmful content (default, suitable for most enterprise use)
- **High** — most permissive: only blocks severely harmful content (use with caution, requires justification)

### 5.2 Adding Custom Blocklist Terms

1. Edit `config/aifoundry/raiblocklist.json` — add organization-specific blocked terms
2. Run `azd provision` to apply

### 5.3 Customizing System Prompt Guardrails

For file-based prompts:
1. Edit files in `prompts/single_agent_rag/` (or your active strategy folder)
2. Redeploy the orchestrator container (`azd deploy`)

For Cosmos-based prompts:
1. Edit prompts directly in Cosmos DB Data Explorer
2. Changes take effect on the next request — no redeployment needed

---

## 6. Monitoring RAI Violations

GPT-RAG feeds content safety events into Application Insights. Use KQL queries to monitor violations.

### Safety Violations by Category and Severity

```kql
customEvents
| where name == "content_safety_violation"
| extend
    category = tostring(customDimensions["category"]),
    severity = tostring(customDimensions["severity"]),
    query = tostring(customDimensions["query"])
| summarize count() by category, severity, bin(timestamp, 1d)
| render barchart
```

### Low-Groundedness Responses (Potential Hallucinations)

```kql
customEvents
| where name == "evaluation_result"
| extend
    query = tostring(customDimensions["query"]),
    groundedness = todouble(customDimensions["groundedness_score"]),
    relevance = todouble(customDimensions["relevance_score"])
| where groundedness < 0.7
| project timestamp, query, groundedness, relevance
| order by groundedness asc
| take 50
```

---

## 7. Evaluation — Azure AI Foundry Evaluators

GPT-RAG stores an evaluation API key in Key Vault during provisioning (step 4 of `setup.py`), enabling integration with Azure AI Foundry's built-in evaluators.

### Risk and Safety Evaluators

| Evaluator | What It Detects |
|-----------|----------------|
| **Hate and Unfairness** | Biased, discriminatory, or hateful content |
| **Sexual** | Inappropriate sexual content |
| **Violence** | Violent content or incitement |
| **Self-Harm** | Content promoting self-harm |
| **Content Safety** | Comprehensive safety assessment (all categories) |
| **Protected Materials** | Unauthorized use of copyrighted content |
| **Code Vulnerability** | Security issues in generated code |
| **Ungrounded Attributes** | Fabricated information inferred from user interactions |

### Quality Evaluators (RAI-Adjacent)

| Evaluator | What It Measures |
|-----------|-----------------|
| **Groundedness** | Is the response supported by the retrieved context? |
| **Groundedness Pro** (preview) | Enhanced groundedness measurement |
| **Relevance** | Is the response relevant to the query? |
| **Retrieval** | How effectively does the system retrieve relevant information? |
| **Response Completeness** | Is the response complete vs ground truth? |

### Recommended Evaluator Combinations for GPT-RAG

| Application Type | Recommended Evaluators |
|-----------------|----------------------|
| RAG applications | Retrieval + Groundedness + Relevance + Content Safety |
| Agent applications | Tool Call Accuracy + Task Adherence + Intent Resolution + Content Safety |
| All applications | Add risk/safety evaluators (Hate, Sexual, Violence, Self-Harm) |

### Production Evaluation Workflow

1. Configure sampling rate (e.g., evaluate 10% of production traffic) in Foundry portal
2. Select evaluators to run
3. Results appear in the Monitoring dashboard
4. Published to Application Insights for alerting

**Note:** GPT-RAG does not automate the evaluation integration out of the box. The user feedback loop (v2.1.0+, thumbs up/down → Application Insights) provides the raw data needed to build evaluation pipelines manually.

---

## 8. User Feedback Loop (v2.1.0+)

The Frontend collects user feedback and sends events to Application Insights, providing data for RAI monitoring and evaluation.

| Configuration Key | Default | Purpose |
|-------------------|---------|---------|
| `ENABLE_USER_FEEDBACK` | `false` | Show feedback buttons (thumbs up/down) |
| `USER_FEEDBACK_RATING` | `false` | Enable 5-star rating popup with comments |

This data can be used to:
- Monitor response quality over time
- Identify problematic queries or topics
- Feed into Azure AI Foundry evaluation workflows
- Track improvement after prompt or model changes

---

## 9. Troubleshooting

| Problem | Likely Cause | Resolution |
|---------|-------------|------------|
| RAI content blocked unexpectedly | Content filter threshold too aggressive | Adjust severity thresholds in `raipolicies.json`, re-run `azd provision` |
| Blocklist not applied | `setup.py` failed silently during postprovision | Check postprovision logs, verify `CognitiveServicesContributor` role on deployer |
| Policy not associated with deployment | Step 3 of `setup.py` skipped embedding models | Expected behavior — embedding models don't need content filtering |
| System prompt changes not taking effect | Using `PROMPT_SOURCE=file` (default) | Redeploy orchestrator container, or switch to `PROMPT_SOURCE=cosmos` |
| No content safety events in App Insights | Violations not being logged | Verify Application Insights connection string in orchestrator config |
| RAI policies command returns empty | Wrong resource or account name | Verify Cognitive Services account name matches the one provisioned by Bicep |

### Diagnostic Commands

**List all RAI policies on the account:**
```bash
az cognitiveservices account list-rai-policies \
  --name {cognitive-services-account-name} \
  --resource-group {rg-name}
```

---

## 10. Key Configuration Files Reference

| File | Path | Purpose |
|------|------|---------|
| `raiblocklist.json` | `config/aifoundry/` | Custom content blocklist definitions |
| `raipolicies.json` | `config/aifoundry/` | Content filtering policy severity thresholds |
| `setup.py` | `config/aifoundry/` | RAI automation script (runs in postprovision) |
| `main.txt` | `prompts/single_agent_rag/` | System prompt for default strategy |
| `main.txt` | `prompts/mcp/` | System prompt for MCP strategy |
| `triage_agent.txt` | `prompts/nl2sql/` | System prompt for NL2SQL triage agent |

---

## 11. Production Readiness Questions

These are open questions that should be resolved before going to production:

1. **Alert mechanism** — When content is blocked by RAI filters, is there an automated alert? Default: only Application Insights logging, no proactive alerting out of the box.
2. **Audit trail** — How do we audit what was filtered and why? Is there a log of blocked content with reason and category?
3. **Industry-specific thresholds** — For regulated industries (healthcare, finance), are the default `raipolicies.json` thresholds sufficient, or do we need industry-specific templates?
4. **Blocklist management** — Can we integrate the RAI blocklist with an existing compliance team's term management system?
5. **Tool call interaction** — How does content filtering interact with the agent's tool calls (MCP strategy)? Are tool inputs/outputs also filtered?
6. **GA timeline** — Some filters (jailbreak, indirect attack, protected material) may have different availability status. Verify GA status for your region.

---

## 12. Defense-in-Depth Summary

```
┌──────────────────────────────────────────────────────────────┐
│                     AZURE OPENAI DEPLOYMENT                   │
│                                                              │
│  ┌────────────────┐  ┌───────────────────────────────────┐  │
│  │  Blocklists    │  │  Content Filtering Policies        │  │
│  │  (hard block)  │  │  Hate │ Sexual │ Violence │ Self-  │  │
│  │                │  │  Harm │ Jailbreak │ Indirect Attack │  │
│  └────────────────┘  └───────────────────────────────────┘  │
│                                                              │
│  Applied to: chat models only (not embeddings)               │
│  Configured by: setup.py during postprovision                │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR AGENT                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  System Prompt Guardrails                              │  │
│  │  Persona │ Domain boundaries │ Refusal behavior        │  │
│  │  Citation rules │ Output format constraints             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Source: file (default) or Cosmos DB (runtime editing)       │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     AZURE AI SEARCH                           │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Document-Level Security                               │  │
│  │  ACL fields │ OBO token │ Permission filters            │  │
│  │  No token = no results (fail-closed)                    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Populated by: ingestion pipeline (indexers)                 │
│  Enforced by: AI Search security filters at query time       │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     MONITORING & EVALUATION                   │
│                                                              │
│  Application Insights │ User Feedback │ Foundry Evaluators   │
│  KQL queries │ Safety violation tracking │ Groundedness       │
└──────────────────────────────────────────────────────────────┘
```

---

## 13. Post-Deployment RAI Checklist for the Implementation Team

After the accelerator is deployed with `azd provision` + `azd deploy`, the default RAI configuration is functional but generic. The following items must be addressed by the implementation team to make it production-ready for the specific organization.

### Priority 1 — Must Do Before Go-Live

| # | Task | Where to Configure | What to Do |
|---|------|--------------------|------------|
| 1 | **Customize the blocklist** | `config/aifoundry/raiblocklist.json` | Add organization-specific blocked terms: competitor names, internal project codenames, confidential product names, employee names that shouldn't appear in responses. Re-run `azd provision` to apply. |
| 2 | **Review content filter thresholds** | `config/aifoundry/raipolicies.json` | Evaluate whether the default `medium` severity for all four categories (hate, sexual, violence, self-harm) is appropriate for your use case. Regulated industries (healthcare, finance, education) may need `low` (stricter). Re-run `azd provision` to apply. |
| 3 | **Write the system prompt** | `prompts/single_agent_rag/main.txt` (or your active strategy folder) | Replace the default system prompt with organization-specific instructions: persona, tone, domain boundaries (what topics the agent should refuse), citation format, language requirements, disclaimer text. Redeploy the orchestrator container. |
| 4 | **Verify `ALLOW_ANONYMOUS=false`** | App Configuration | Ensure anonymous queries are blocked in production. This is the default but must be explicitly verified — if set to `true`, document-level security is completely bypassed. |
| 5 | **Test with real user permissions** | End-to-end testing | Log in as users with different SharePoint/blob permissions and verify they see different search results. A user without access to a document should never see its content in responses. |

### Priority 2 — Should Do Within First Week

| # | Task | Where to Configure | What to Do |
|---|------|--------------------|------------|
| 6 | **Enable user feedback** | App Configuration: `ENABLE_USER_FEEDBACK=true` | Turn on thumbs up/down in the UI so users can flag bad responses. Optionally enable `USER_FEEDBACK_RATING=true` for 5-star ratings with comments. |
| 7 | **Set up monitoring alerts** | Application Insights → Alerts | Create alert rules for content safety violations (KQL: `customEvents \| where name == "content_safety_violation"`). Set thresholds for notification — e.g., alert if more than 10 violations per hour. |
| 8 | **Switch to Cosmos-based prompts** | App Configuration: `PROMPT_SOURCE=cosmos` | Enable runtime prompt editing so the team can iterate on system prompt guardrails without redeploying the container. Run `upload_prompts.py` to seed initial prompts, then edit in Cosmos DB Data Explorer. |
| 9 | **Verify jailbreak + indirect attack filters** | Azure portal → Cognitive Services → Content Safety | Confirm that jailbreak detection and indirect attack protection are enabled on the chat model deployment. These are binary (on/off) and may not be covered by `raipolicies.json` — verify via portal or REST API. |
| 10 | **Document the RAI policy for stakeholders** | Internal wiki / compliance docs | Create an internal document explaining what content filters are in place, what the blocklist contains, and how the system prompt constrains behavior. Compliance and legal teams will need this for sign-off. |

### Priority 3 — Should Do Before Production Scale

| # | Task | Where to Configure | What to Do |
|---|------|--------------------|------------|
| 11 | **Set up Foundry evaluation pipeline** | Azure AI Foundry portal → Evaluations | Configure continuous evaluation with at least: Groundedness + Relevance + Content Safety evaluators. Sample 10% of production traffic. Results will flow to Application Insights for alerting. |
| 12 | **Build a KQL monitoring dashboard** | Application Insights → Workbooks | Create a dashboard combining: safety violation trends, groundedness scores over time, user feedback sentiment, blocked content categories. Share with the operations team. |
| 13 | **Red-team the system prompt** | Manual testing | Attempt prompt injection, jailbreak, and out-of-scope queries against the deployed system. Document findings and iterate on system prompt guardrails. |
| 14 | **Review protected material filter** | Azure portal → Content Safety | Verify that the protected material filter is active — this prevents the model from reproducing copyrighted content. Important for legal compliance. |
| 15 | **Establish a blocklist review cadence** | Team process | Define who owns the blocklist, how often it's reviewed (monthly recommended), and the process for adding/removing terms. Re-run `azd provision` after each update. |

### Quick Reference — Where Everything Lives

| What | File / Location | How to Update |
|------|----------------|---------------|
| Blocklist terms | `config/aifoundry/raiblocklist.json` | Edit JSON → `azd provision` |
| Filter thresholds | `config/aifoundry/raipolicies.json` | Edit JSON → `azd provision` |
| System prompt (file) | `prompts/{strategy}/main.txt` | Edit file → `azd deploy` (redeploy orchestrator) |
| System prompt (Cosmos) | Cosmos DB → `prompts` container | Edit in Data Explorer → takes effect on next request |
| User feedback toggle | App Configuration → `ENABLE_USER_FEEDBACK` | Change value → container picks up on next restart |
| Anonymous access | App Configuration → `ALLOW_ANONYMOUS` | Change value → container picks up on next restart |
| Monitoring alerts | Application Insights → Alerts | Configure in Azure portal |
| Evaluation pipeline | Azure AI Foundry portal → Evaluations | Configure in Foundry portal |
