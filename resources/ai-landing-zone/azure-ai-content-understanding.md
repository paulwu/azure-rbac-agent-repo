# Azure AI Content Understanding

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.CognitiveServices` |
| **Resource Type** | `Microsoft.CognitiveServices/accounts` |
| **Azure Portal Category** | AI + Machine Learning > Content Understanding |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/ai-services/content-understanding/overview) |
| **Pricing** | [Content Understanding Pricing](https://azure.microsoft.com/pricing/details/cognitive-services/) |
| **SLA** | Preview — no production SLA |

## Overview

Azure AI Content Understanding *(preview)* is a multimodal AI service that analyzes and extracts structured information from documents, images, audio, and video content through a unified API. In an AI Landing Zone it provides a single content processing layer that combines OCR, vision, speech, and language capabilities, reducing the need to orchestrate multiple individual AI services for end-to-end content extraction and understanding pipelines.

## Least-Privilege RBAC Reference

> Content Understanding uses the standard Cognitive Services RBAC model, separating **management plane** (resource lifecycle) from **data plane** (content analysis). As a preview service, role support may evolve before GA.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Content Understanding resource | Resource Group | `Cognitive Services Contributor` | Creates a Cognitive Services account. |
| Create an analyzer (content processing definition) | Content Understanding resource | `Cognitive Services Contributor` | Analyzers define the extraction schema and processing pipeline. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update resource network settings | Content Understanding resource | `Cognitive Services Contributor` | |
| Update analyzer definition | Content Understanding resource | `Cognitive Services Contributor` | Modify extraction fields or processing configuration. |
| Modify resource tags | Content Understanding resource | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Content Understanding resource | Resource Group | `Cognitive Services Contributor` | |
| Delete an analyzer | Content Understanding resource | `Cognitive Services Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Analyze content (run analyzer on documents/images/audio/video) | Content Understanding resource | `Cognitive Services User` | Minimum role for inference. Supports all content modalities. |
| Get analysis results | Content Understanding resource | `Cognitive Services User` | Retrieve structured extraction results. |
| List analyzers | Content Understanding resource | `Cognitive Services User` | View available analyzer configurations. |
| Read resource keys (API key auth) | Content Understanding resource | `Cognitive Services Contributor` | Key listing grants equivalent data access — prefer Entra ID auth. |
| Configure Private Endpoint | Content Understanding resource + VNet | `Cognitive Services Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Content Understanding resource | `Monitoring Contributor` | |
| Access source content in Blob Storage | Storage Account | `Storage Blob Data Reader` | Required when Content Understanding reads input content from Azure Blob Storage. |

## Supported Content Types

| Content Type | Capabilities | Minimum Role |
|---|---|---|
| **Documents** (PDF, Word, etc.) | Text extraction, field extraction, classification | `Cognitive Services User` |
| **Images** (JPEG, PNG, TIFF) | OCR, object detection, visual analysis | `Cognitive Services User` |
| **Audio** (WAV, MP3, etc.) | Transcription, speaker identification, summarization | `Cognitive Services User` |
| **Video** (MP4, etc.) | Scene detection, transcription, visual analysis | `Cognitive Services User` |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores input documents, images, audio, or video files that Content Understanding reads from Azure Blob Storage during analysis. | Optional (inline submission supported) |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores API keys when Entra ID authentication is not used; accessed at runtime by consuming applications. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs and metrics via Diagnostic Settings for monitoring analysis requests and service health. | Optional |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to prevent sensitive content from traversing the public internet. | Optional (recommended) |

## Notes / Considerations

- **Preview service** — Azure AI Content Understanding is currently in preview. Role support, API surface, and capabilities may change before GA.
- **`Cognitive Services User`** is the minimum role for all content analysis operations — assign to application managed identities.
- **Disable API key authentication** (`disableLocalAuth: true`) and use Entra ID token-based auth for production-bound workloads even during preview.
- Content Understanding combines capabilities from Document Intelligence, Vision, Speech, and Language services into a **single unified API** — reducing orchestration complexity.
- **Analyzers** are the core configuration primitive — they define what fields and structure to extract from content.
- **Managed Identity** is the preferred authentication pattern for applications consuming Content Understanding APIs.
- **Private Endpoint** should be configured for sensitive content processing to prevent data from traversing the public internet.
- Input content can be provided inline or referenced from Azure Blob Storage — ensure the Content Understanding resource's managed identity has `Storage Blob Data Reader` if accessing storage directly.
- Consider using **Azure AI Document Intelligence** for production document extraction workloads that require GA-level SLA guarantees while Content Understanding remains in preview.

## Related Resources

- [Azure AI Document Intelligence](./azure-ai-document-intelligence.md) — Dedicated document extraction service (GA)
- [Azure AI Services](./azure-ai-services.md) — Parent service family
- [Azure Applied AI Services](./azure-applied-ai-services.md) — Applied AI services overview
- [Azure OpenAI](./azure-openai.md) — LLM processing of extracted content
- [Azure AI Search](./azure-ai-search.md) — Index extracted content for retrieval and RAG
- [Azure AI Foundry](./azure-ai-foundry.md) — Orchestration platform for AI workflows
- [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) — Source content storage
