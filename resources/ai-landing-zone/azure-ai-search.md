# Azure AI Search (Cognitive Search)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Search` |
| **Resource Type** | `Microsoft.Search/searchServices` |
| **Azure Portal Category** | AI + Machine Learning > AI Search |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/search/search-what-is-azure-search) |
| **Pricing** | [AI Search Pricing](https://azure.microsoft.com/pricing/details/search/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/search/) |

## Overview

Azure AI Search is a fully managed search-as-a-service platform providing full-text search, vector search, semantic ranking, and RAG (Retrieval Augmented Generation) capabilities. In an AI Landing Zone it is the knowledge retrieval layer for Azure OpenAI applications, indexing documents from Blob Storage, ADLS Gen2, SharePoint, and databases.

## Least-Privilege RBAC Reference

> AI Search separates **management plane** (service configuration) from **data plane** (index and document access). Data plane roles are critical for controlling who can read or modify search indexes.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create AI Search Service | Resource Group | `Search Service Contributor` | Management-plane role for creating the service. |
| Create an Index (schema definition) | Search Service | `Search Index Data Contributor` | Data-plane role. Allows index creation, update, and data management. |
| Create an Indexer (data ingestion pipeline) | Search Service | `Search Index Data Contributor` | Indexers pull data from Azure Blob, SQL, Cosmos DB, etc. |
| Create a Data Source (connection to source) | Search Service | `Search Index Data Contributor` | |
| Create a Skillset (AI enrichment pipeline) | Search Service | `Search Index Data Contributor` | |
| Create a Knowledge Store | Search Service + Storage | `Search Index Data Contributor` + `Storage Blob Data Contributor` | Skillset outputs written to storage. |
| Create an Alias | Search Service | `Search Index Data Contributor` | Index aliases for zero-downtime index swaps. |
| Create a Semantic Configuration | Search Service | `Search Index Data Contributor` | Configures semantic ranking for an index. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update index schema | Search Service | `Search Index Data Contributor` | Some schema changes require index rebuild. |
| Update/re-run indexer | Search Service | `Search Index Data Contributor` | |
| Modify service capacity (replicas, partitions) | Search Service | `Search Service Contributor` | |
| Update skillset | Search Service | `Search Index Data Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete documents from index | Search Service | `Search Index Data Contributor` | |
| Delete an index | Search Service | `Search Index Data Contributor` | Permanently removes all documents. |
| Delete an indexer / data source / skillset | Search Service | `Search Index Data Contributor` | |
| Delete the Search Service | Resource Group | `Search Service Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Query an index (search/suggest/autocomplete) | Search Service | `Search Index Data Reader` | Minimum role for application search queries. |
| Vector search / semantic search queries | Search Service | `Search Index Data Reader` | Same role for all query types. |
| Upload documents to index | Search Service | `Search Index Data Contributor` | Push-mode document indexing. |
| Read index schema (for integration) | Search Service | `Search Index Data Reader` | |
| Configure network rules / Private Endpoint | Search Service | `Search Service Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Search Service | `Monitoring Contributor` | |
| Manage API keys (list/regenerate) | Search Service | `Search Service Contributor` | API keys grant equivalent data access — prefer Entra ID auth. |

## Data Plane Role Summary

| Role | Query Index | Upload Documents | Create/Delete Index | Manage Service |
|---|---|---|---|---|
| `Search Index Data Contributor` | ✅ | ✅ | ✅ | ❌ |
| `Search Index Data Reader` | ✅ | ❌ | ❌ | ❌ |
| `Search Service Contributor` | ❌ | ❌ | ❌ | ✅ |

## Common Assignment Patterns

| Principal | Role | Scope | Purpose |
|---|---|---|---|
| Application querying search | `Search Index Data Reader` | Search Service | Read-only search queries for UI/API |
| Azure OpenAI resource (RAG) | `Search Index Data Reader` | Search Service | OpenAI On Your Data queries the index |
| Indexing pipeline identity (ADF, AML) | `Search Index Data Contributor` | Search Service | Populate and maintain indexes |
| Indexer managed identity (pull-mode) | `Storage Blob Data Reader` | Source storage | Indexer reads source documents |
| Platform team | `Search Service Contributor` | Search Service | Manage service capacity and network config |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Indexers pull source documents from Blob containers; the Search service's managed identity requires `Storage Blob Data Reader` on the source storage account. | Optional (required for pull-mode indexing) |
| [Azure Data Lake Storage Gen2](../data-landing-zone/azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | ADLS Gen2 data source for indexers processing large document corpora; Search managed identity requires `Storage Blob Data Reader`. | Optional |
| [Azure AI Services](./azure-ai-services.md) | `Microsoft.CognitiveServices/accounts` | Integrated vectorization and AI enrichment skillsets use Azure AI Services for OCR, entity extraction, and embedding generation; requires `Cognitive Services User` on the AI Services resource. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs (indexer execution, query traffic, throttling) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity (`privatelink.search.windows.net`) to isolate the search service from public internet access. | Optional (strongly recommended) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.search.windows.net` for Private Endpoint-connected clients and indexer connections. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **`Search Index Data Reader`** is the minimum for application search — assign to all query-only service principals and managed identities.
- **Disable API key authentication** (`disableLocalAuth: true`) and require Entra ID token auth in production.
- **Indexer managed identity** needs read access on the source (e.g., `Storage Blob Data Reader` on the storage account, `Reader` on Cosmos DB with data reader role).
- **Azure OpenAI On Your Data** connects to AI Search using the OpenAI resource's managed identity — assign `Search Index Data Reader` to the OpenAI managed identity.
- **Semantic search** and **vector search** require specific index configurations — both are available to `Search Index Data Reader` at query time.
- **Private Endpoints**: Three separate endpoints are available (portal, index, indexer) — configure all three for fully private deployments.
- **Integrated vectorization** (built-in embedding skill) calls Azure OpenAI or AI Services automatically — configure connection using managed identity.

## Related Resources

- [Azure OpenAI](./azure-openai.md) — RAG architecture: OpenAI + AI Search
- [Azure AI Services](./azure-ai-services.md) — Integrated vectorization using embedding models
- [Azure AI Document Intelligence](./azure-ai-document-intelligence.md) — Extract structured data from documents for indexing
- [Azure AI Content Understanding](./azure-ai-content-understanding.md) — Multimodal content extraction for indexing
- [Azure Data Lake Storage Gen2](../data-landing-zone/azure-data-lake-storage-gen2.md) — Document source for indexers
- [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) — `privatelink.search.windows.net`
