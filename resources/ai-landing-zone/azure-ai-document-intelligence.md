# Azure AI Document Intelligence

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.CognitiveServices` |
| **Resource Type** | `Microsoft.CognitiveServices/accounts` (kind: `FormRecognizer`) |
| **Azure Portal Category** | AI + Machine Learning > Document Intelligence |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/ai-services/document-intelligence/overview) |
| **Pricing** | [Document Intelligence Pricing](https://azure.microsoft.com/pricing/details/ai-document-intelligence/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/cognitive-services/) |

## Overview

Azure AI Document Intelligence (formerly Form Recognizer) is a cloud-based AI service that extracts text, key-value pairs, tables, and structured data from documents using pre-built and custom models. In an AI Landing Zone it serves as the document processing backbone for intelligent document workflows, converting unstructured PDFs, invoices, receipts, ID documents, and forms into structured, queryable data for downstream AI pipelines.

## Least-Privilege RBAC Reference

> Document Intelligence separates **management plane** (resource lifecycle and configuration) from **data plane** (document analysis and model training). Use `Cognitive Services User` for inference-only workloads.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Document Intelligence resource | Resource Group | `Cognitive Services Contributor` | Creates a `FormRecognizer` kind account under `Microsoft.CognitiveServices`. |
| Build a custom extraction model | Document Intelligence resource | `Cognitive Services Contributor` | Requires training data in Azure Blob Storage. |
| Build a custom classifier | Document Intelligence resource | `Cognitive Services Contributor` | Custom classifiers categorize document types before extraction. |
| Compose a custom model | Document Intelligence resource | `Cognitive Services Contributor` | Combines multiple custom models into a single model endpoint. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update resource network settings | Document Intelligence resource | `Cognitive Services Contributor` | |
| Retrain custom model | Document Intelligence resource | `Cognitive Services Contributor` | Requires access to updated training data in Blob Storage. |
| Modify resource tags | Document Intelligence resource | `Tag Contributor` | Tags only; no other resource changes. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Document Intelligence resource | Resource Group | `Cognitive Services Contributor` | |
| Delete a custom model | Document Intelligence resource | `Cognitive Services Contributor` | Removes model; does not affect previously analyzed documents. |
| Delete a custom classifier | Document Intelligence resource | `Cognitive Services Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Analyze documents (pre-built models: invoice, receipt, ID) | Document Intelligence resource | `Cognitive Services User` | Minimum role for application inference. |
| Analyze documents (custom models) | Document Intelligence resource | `Cognitive Services User` | Same role for pre-built and custom model inference. |
| Analyze documents (layout extraction) | Document Intelligence resource | `Cognitive Services User` | Layout API extracts text, tables, selection marks, and structure. |
| Analyze documents (read / OCR) | Document Intelligence resource | `Cognitive Services User` | Read API extracts printed and handwritten text. |
| Copy custom model between resources | Source + Target resource | `Cognitive Services Contributor` on both | Cross-resource model copy for environment promotion (dev → prod). |
| List/get custom models | Document Intelligence resource | `Cognitive Services User` | Read model metadata for integration. |
| Read resource keys (API key auth) | Document Intelligence resource | `Cognitive Services Contributor` | Key listing grants equivalent data access — prefer Entra ID auth. |
| Configure Private Endpoint | Document Intelligence resource + VNet | `Cognitive Services Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Document Intelligence resource | `Monitoring Contributor` | |
| Access training data in Blob Storage | Storage Account | `Storage Blob Data Reader` | Managed identity of Document Intelligence or the training principal needs read access to training data containers. |

## Pre-Built Model Summary

| Model | Use Case | Minimum Role |
|---|---|---|
| **Read** | OCR — extract printed and handwritten text | `Cognitive Services User` |
| **Layout** | Tables, structure, selection marks | `Cognitive Services User` |
| **General Document** | Key-value pairs from any document | `Cognitive Services User` |
| **Invoice** | Structured invoice field extraction | `Cognitive Services User` |
| **Receipt** | Structured receipt field extraction | `Cognitive Services User` |
| **ID Document** | Identity document field extraction | `Cognitive Services User` |
| **W-2 / Tax** | US tax form extraction | `Cognitive Services User` |
| **Health Insurance Card** | Insurance card field extraction | `Cognitive Services User` |
| **Contract** | Contract clause and field extraction | `Cognitive Services User` |
| **Custom** | User-trained extraction model | `Cognitive Services User` (inference) / `Cognitive Services Contributor` (training) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores training documents and labeled data for custom model training; required when building or retraining custom or composed models. | Required (custom models) / Optional (pre-built models) |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores API keys and endpoint URLs when Entra ID auth is not used; also used by consuming applications to retrieve credentials at runtime. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs and metrics via Diagnostic Settings for monitoring analysis requests and errors. | Optional |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to prevent document content from traversing the public internet. | Optional (recommended) |

## Notes / Considerations

- **`Cognitive Services User`** is the minimum role for all document analysis (inference) operations — assign to application managed identities.
- **Disable API key authentication** (`disableLocalAuth: true`) and use Entra ID token-based auth for all production deployments.
- **Custom model training** requires training data stored in Azure Blob Storage — the training principal needs `Storage Blob Data Reader` on the storage container.
- **Composed models** combine multiple custom models into a single model ID — useful for multi-document-type scenarios where a classifier selects the appropriate sub-model.
- **Model copy** between resources requires `Cognitive Services Contributor` on both source and target — use for promoting models from dev to prod environments.
- **Private Endpoint** should be configured in AI Landing Zones to prevent document content from traversing the public internet.
- **Document Intelligence Studio** (web UI) requires the user to have `Cognitive Services Contributor` for training and `Cognitive Services User` for analysis.
- Use **managed identity** for Document Intelligence resources that access Blob Storage for training data rather than SAS tokens or storage keys.
- Document Intelligence is also available as part of a **multi-service AI Services account** (kind: `CognitiveServices`) — dedicated single-service accounts provide better role isolation and quota management.

## Related Resources

- [Azure AI Services](./azure-ai-services.md) — Parent service family; Document Intelligence is a Cognitive Services account
- [Azure Applied AI Services](./azure-applied-ai-services.md) — Broader applied AI service context
- [Azure AI Content Understanding](./azure-ai-content-understanding.md) — Multimodal content analysis including documents
- [Azure AI Search](./azure-ai-search.md) — Index extracted document content for search and RAG
- [Azure OpenAI](./azure-openai.md) — LLM processing of extracted document data
- [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) — Training data and document source storage
- [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) — Store endpoint URLs and fallback keys
