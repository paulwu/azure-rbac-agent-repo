# Azure Applied AI Services

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.CognitiveServices` |
| **Resource Types** | `Microsoft.CognitiveServices/accounts` (various kinds), `Microsoft.VideoIndexer/accounts` |
| **Azure Portal Category** | AI + Machine Learning |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/applied-ai-services/) |
| **Pricing** | Per-service pricing pages |

## Overview

Azure Applied AI Services are higher-level, scenario-specific AI services built on top of Azure AI Services. They include:

| Service | Resource Kind | Primary Use Case |
|---|---|---|
| **Azure AI Document Intelligence** (Form Recognizer) | `FormRecognizer` | Extract structured data from documents, forms, invoices |
| **Azure AI Video Indexer** | `Microsoft.VideoIndexer/accounts` | Index and search video/audio content |
| **Azure AI Metrics Advisor** | `MetricsAdvisor` | Anomaly detection on time-series metrics |
| **Azure AI Immersive Reader** | `ImmersiveReader` | Reading assistance for accessibility |
| **Azure AI Anomaly Detector** | `AnomalyDetector` | API-based anomaly detection |
| **Azure Health Data Services** | `HealthDataAIServices` | Healthcare AI (DICOM, FHIR, MedTech) |

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Document Intelligence resource | Resource Group | `Cognitive Services Contributor` | |
| Create Metrics Advisor resource | Resource Group | `Cognitive Services Contributor` | |
| Create Anomaly Detector resource | Resource Group | `Cognitive Services Contributor` | |
| Create Immersive Reader resource | Resource Group | `Cognitive Services Contributor` | |
| Create Video Indexer account | Resource Group | `Contributor` | Video Indexer uses a different resource provider (`Microsoft.VideoIndexer`). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify resource network / endpoint settings | AI resource | `Cognitive Services Contributor` | |
| Update Metrics Advisor workspace settings | Metrics Advisor resource | `Cognitive Services Metrics Advisor Administrator` | Workspace-level admin for Metrics Advisor. |
| Update Video Indexer account settings | Video Indexer account | `Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete any Applied AI resource | Resource Group | `Cognitive Services Contributor` (or `Contributor` for Video Indexer) | |

### ⚙️ Configure — Document Intelligence

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Analyze documents (inference) | Document Intelligence resource | `Cognitive Services User` | Minimum for API inference calls. |
| Train custom models | Document Intelligence resource | `Cognitive Services Contributor` | Custom model training requires contributor access. |
| Manage custom models (delete, copy) | Document Intelligence resource | `Cognitive Services Contributor` | |
| Configure network rules / Private Endpoint | Document Intelligence resource | `Cognitive Services Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Document Intelligence resource | `Monitoring Contributor` | |

### ⚙️ Configure — Metrics Advisor

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Use workspace (view anomalies, configure alerts) | Metrics Advisor resource | `Cognitive Services Metrics Advisor User` | Workspace-level user access. |
| Administer workspace (data feeds, hooks) | Metrics Advisor resource | `Cognitive Services Metrics Advisor Administrator` | Full workspace administration. |

### ⚙️ Configure — Video Indexer

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Upload and index videos | Video Indexer account | `Contributor` | No data-plane role separation — `Contributor` is required for all operations. |
| View/search indexed video content | Video Indexer account | `Reader` | View-only access to indexed content. |
| Configure Azure Media Services integration | Video Indexer + AMS account | `Contributor` on both | Video Indexer may use Azure Media Services for encoding (legacy). |

### ⚙️ Configure — Anomaly Detector

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Make API calls (detect anomalies) | Anomaly Detector resource | `Cognitive Services User` | |
| Manage resource settings | Anomaly Detector resource | `Cognitive Services Contributor` | |

### ⚙️ Configure — Immersive Reader

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Use Immersive Reader (application) | Immersive Reader resource | `Cognitive Services Immersive Reader User` | Narrow role specifically for this service. |
| Manage resource settings | Immersive Reader resource | `Cognitive Services Contributor` | |

## Role Summary by Service

| Service | Use/Inference Role | Manage/Train Role |
|---|---|---|
| Document Intelligence | `Cognitive Services User` | `Cognitive Services Contributor` |
| Anomaly Detector | `Cognitive Services User` | `Cognitive Services Contributor` |
| Metrics Advisor (workspace) | `Cognitive Services Metrics Advisor User` | `Cognitive Services Metrics Advisor Administrator` |
| Immersive Reader | `Cognitive Services Immersive Reader User` | `Cognitive Services Contributor` |
| Video Indexer | `Reader` (view) | `Contributor` (manage/upload) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Document Intelligence custom model training reads labeled training documents from Blob Storage; the resource's managed identity requires `Storage Blob Data Reader` on the training data container. | Required (Document Intelligence custom models) |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores API keys for consuming applications; `Key Vault Secrets User` required for the application's managed identity when key references are used. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs and metrics via Diagnostic Settings for monitoring API usage and model training operations. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity for Document Intelligence and Anomaly Detector to restrict access to the private network. | Optional (strongly recommended) |

## Notes / Considerations

- **Document Intelligence** custom models require training data in Azure Blob Storage — the resource's managed identity or service principal needs `Storage Blob Data Reader` on the training data container.
- **Metrics Advisor** has its own workspace RBAC system (`Metrics Advisor Administrator` / `Metrics Advisor User`) that maps to workspace access control, separate from Azure RBAC.
- **Disable API key authentication** and use Entra ID token auth (`Cognitive Services User`) for all production inference workloads.
- **Azure Health Data Services** (FHIR, DICOM, MedTech) has additional service-specific data plane roles — see the [Health Data Services documentation](https://learn.microsoft.com/azure/health-data-services/) for details.
- Video Indexer does **not** currently support Entra ID data-plane roles for video API calls — use account-level access tokens for API authentication.

## Related Resources

- [Azure AI Services](./azure-ai-services.md) — Underlying AI building blocks
- [Azure AI Document Intelligence](./azure-ai-document-intelligence.md) — Dedicated document extraction resource (detailed reference)
- [Azure AI Content Understanding](./azure-ai-content-understanding.md) — Multimodal content analysis (preview)
- [Azure OpenAI](./azure-openai.md) — LLM-based document understanding
- [Azure AI Search](./azure-ai-search.md) — Indexing and searching processed document content
- [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) — Document Intelligence training data storage
