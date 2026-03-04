# Grounding with Bing Search

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Bing` |
| **Resource Type** | `Microsoft.Bing/accounts` |
| **Azure Portal Category** | AI + Machine Learning > Grounding with Bing Search |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/ai-services/agents/how-to/tools/bing-grounding) |
| **Pricing** | [Bing Search Pricing](https://www.microsoft.com/en-us/bing/apis/pricing) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/bing-entity-search-api/) |

## Overview

Grounding with Bing Search is an Azure resource that enables Azure OpenAI and Azure AI Foundry agents to ground large language model (LLM) responses with real-time web search results from Bing. In an AI Landing Zone it provides the web knowledge retrieval layer for AI applications that need up-to-date information beyond their training data cutoff, improving factual accuracy and reducing hallucinations in generative AI outputs.

## Least-Privilege RBAC Reference

> Grounding with Bing Search uses the `Microsoft.Bing` resource provider. Management plane operations use standard Azure RBAC. Data plane access is **API key-based only** — no Entra ID data-plane roles exist for Bing resources.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Grounding with Bing Search resource | Resource Group | `Contributor` | No service-specific built-in role exists for `Microsoft.Bing`; `Contributor` is required. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update resource settings (SKU, tags) | Bing Search resource | `Contributor` | |
| Regenerate API keys | Bing Search resource | `Contributor` | Key regeneration invalidates the previous key — coordinate rotation across all consumers. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Grounding with Bing Search resource | Resource Group | `Contributor` | Removes the resource and invalidates all API keys. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Use Bing grounding with Azure OpenAI | Bing Search resource | API key required | No Entra ID data-plane roles — data plane access requires an API key. |
| Connect to Azure AI Foundry as a grounding tool | Bing Search resource | API key + `Azure AI Developer` on Foundry project | Connection configured in AI Foundry project settings. |
| Use Bing grounding with Azure AI Agents | Bing Search resource | API key required | Agent tool configuration references the Bing resource connection. |
| Read resource keys | Bing Search resource | `Contributor` | Key access is managed through the management plane. |
| View resource configuration | Bing Search resource | `Reader` | Read-only view of resource metadata and settings. |
| Configure Diagnostic Settings | Bing Search resource | `Monitoring Contributor` | |

## Integration Patterns

| Integration | Configuration | Authentication |
|---|---|---|
| **Azure OpenAI** | Add Bing Search as grounding data source | API key stored in connection or Key Vault |
| **Azure AI Foundry Agents** | Add Bing Grounding as a tool in agent definition | API key via Foundry connection |
| **Azure AI Agents Service** | Configure `BingGroundingTool` in agent | API key via agent connection |
| **Custom application** | Call Bing Search API directly, pass results to LLM | API key in `Ocp-Apim-Subscription-Key` header |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure OpenAI](./azure-openai.md) or [Azure AI Foundry](./azure-ai-foundry.md) | `Microsoft.CognitiveServices/accounts` / `Microsoft.MachineLearningServices/workspaces` | The primary consumer service that issues grounding queries to Bing Search at runtime; Grounding with Bing Search has no value without a connected LLM consumer. | Required |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores the Bing Search API key securely; the consuming application or AI Foundry connection retrieves it at runtime using managed identity. | Optional (strongly recommended) |

## Notes / Considerations

- **No Entra ID data-plane roles** — Bing Search resources use API key-based authentication for all data plane operations. No `Microsoft.Bing` data-plane RBAC roles exist.
- **Store API keys in Azure Key Vault** — since Entra ID auth is not supported for the data plane, securely store and retrieve Bing API keys from Key Vault using managed identity.
- **Grounding improves factual accuracy** — connecting Bing Search to Azure OpenAI or AI Agents provides real-time web context, reducing hallucinations for questions requiring current information.
- **Usage is metered** — Bing Search API calls are billed per transaction. Monitor usage to control costs.
- **Content filtering** — Bing Search results are subject to SafeSearch settings. Configure the SafeSearch level appropriate for your use case.
- **No narrow built-in roles** — `Microsoft.Bing` does not have purpose-built management roles. `Contributor` is the minimum for management plane operations. Scope the assignment to the specific Bing resource to limit blast radius.
- **Azure AI Foundry connections** — when using Bing grounding through AI Foundry, the connection stores the API key securely. Ensure the Foundry project has appropriate access controls.
- Create a **separate Bing Search resource per environment** (dev/staging/prod) for key isolation and independent cost tracking.
- **Rate limits** apply per Bing Search resource tier — select the appropriate SKU for your expected query volume.

## Related Resources

- [Azure OpenAI](./azure-openai.md) — Primary consumer of Bing grounding for RAG and chat completions
- [Azure AI Foundry](./azure-ai-foundry.md) — Agent orchestration platform that integrates Bing grounding
- [Azure AI Search](./azure-ai-search.md) — Alternative/complementary grounding source using private data
- [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) — Secure storage for Bing API keys
