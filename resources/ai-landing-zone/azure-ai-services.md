# Azure AI Services (Cognitive Services)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.CognitiveServices` |
| **Resource Type** | `Microsoft.CognitiveServices/accounts` |
| **Azure Portal Category** | AI + Machine Learning > Azure AI Services |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/ai-services/what-are-ai-services) |
| **Pricing** | [AI Services Pricing](https://azure.microsoft.com/pricing/details/cognitive-services/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/cognitive-services/) |

## Overview

Azure AI Services (formerly Cognitive Services) is a family of pre-built AI services for vision, speech, language, and decision capabilities. In an AI Landing Zone these services provide building blocks for AI-powered applications. Key services include: Computer Vision, Face API, Document Intelligence (Form Recognizer), Language Understanding (CLU/LUIS), Text Analytics, Speech Services, Translator, Content Moderator, and Content Safety.

## Least-Privilege RBAC Reference

> Most Azure AI Services share the `Cognitive Services Contributor` and `Cognitive Services User` roles. Individual service families have additional purpose-built roles.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create an AI Services resource (multi-service) | Resource Group | `Cognitive Services Contributor` | One account for all AI Services APIs. |
| Create a single-service resource (e.g., Speech, Vision) | Resource Group | `Cognitive Services Contributor` | Dedicated accounts for cost/quota separation. |
| Deploy a Custom Vision model | AI Services resource | `Cognitive Services Custom Vision Contributor` | Train and publish custom image classification models. |
| Train a LUIS / CLU model | AI Services resource | `Cognitive Services LUIS Owner` / `Cognitive Services Language Owner` | |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update resource network settings | AI Services resource | `Cognitive Services Contributor` | |
| Update Custom Vision model | AI Services resource | `Cognitive Services Custom Vision Contributor` | |
| Update LUIS application | AI Services resource | `Cognitive Services LUIS Owner` | |
| Update Language model / orchestration | AI Services resource | `Cognitive Services Language Owner` | |
| Configure content filtering (Content Safety) | AI Services resource | `Cognitive Services Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete AI Services resource | Resource Group | `Cognitive Services Contributor` | |
| Delete Custom Vision project | AI Services resource | `Cognitive Services Custom Vision Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Make API calls (generic AI Services) | AI Services resource | `Cognitive Services User` | Minimum role for application inference. |
| Use Speech-to-text / Text-to-speech API | AI Services resource | `Cognitive Services Speech User` | Narrower role for Speech services only. |
| Manage Speech resources (custom speech, training) | AI Services resource | `Cognitive Services Speech Contributor` | |
| Use Language / Text Analytics API | AI Services resource | `Cognitive Services Language Reader` | Read / inference only. |
| Manage Language resources (train, publish) | AI Services resource | `Cognitive Services Language Writer` | Create and update language models. |
| Use Custom Vision for inference (prediction) | AI Services resource | `Cognitive Services Custom Vision Reader` | Prediction endpoint calls only. |
| Train Custom Vision model | AI Services resource | `Cognitive Services Custom Vision Trainer` | Training only; cannot publish. |
| Manage Custom Vision (train + publish) | AI Services resource | `Cognitive Services Custom Vision Contributor` | |
| Use Face API | AI Services resource | `Cognitive Services Face Recognizer` | Face detect/identify — note: Limited Access feature requires approved use case. |
| Use Immersive Reader | AI Services resource | `Cognitive Services Immersive Reader User` | |
| Read keys / endpoint (management plane) | AI Services resource | `Cognitive Services Contributor` | Key listing grants data plane equivalent access — prefer Entra ID auth. |
| Configure Private Endpoint | AI Services resource + VNet | `Cognitive Services Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | AI Services resource | `Monitoring Contributor` | |

## Service-Specific Role Summary

| Service | Use (Read/Inference) | Manage (Create/Train/Publish) |
|---|---|---|
| **All AI Services** | `Cognitive Services User` | `Cognitive Services Contributor` |
| **Speech** | `Cognitive Services Speech User` | `Cognitive Services Speech Contributor` |
| **Language / CLU** | `Cognitive Services Language Reader` | `Cognitive Services Language Writer` / `Language Owner` |
| **LUIS** | `Cognitive Services LUIS Reader` | `Cognitive Services LUIS Writer` / `LUIS Owner` |
| **Custom Vision** | `Cognitive Services Custom Vision Reader` | `Cognitive Services Custom Vision Contributor` |
| **Face API** | `Cognitive Services Face Recognizer` | `Cognitive Services Contributor` |
| **Immersive Reader** | `Cognitive Services Immersive Reader User` | `Cognitive Services Contributor` |
| **Metrics Advisor** | `Cognitive Services Metrics Advisor User` | `Cognitive Services Metrics Advisor Administrator` |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Required by Document Intelligence (Form Recognizer kind) and Custom Vision for training data; service managed identity requires `Storage Blob Data Reader` on training data containers. | Optional (required for custom model training) |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores API keys and endpoint URLs for consuming applications when key-based auth is used; `Key Vault Secrets User` required on vault for the consuming application's managed identity. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs and request metrics via Diagnostic Settings for monitoring API usage and service health. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict AI Services API access to the private network. | Optional (strongly recommended) |

## Notes / Considerations

- **`Cognitive Services User`** is the catch-all inference role — use service-specific roles (Speech User, Language Reader, etc.) for finer grained control when single-service accounts are deployed.
- **Disable API key authentication** (`disableLocalAuth: true`) and use Entra ID token-based auth for all production clients.
- **Face API, Speaker Recognition, and Custom Neural Voice** are **Limited Access** features — require Microsoft approval before use.
- **Content Safety** service enforces content filtering for text and images — configure blocklists and severity thresholds via `Cognitive Services Contributor`.
- **Multi-service account** (kind: `CognitiveServices`) gives access to all services under one endpoint/key — use single-service accounts in production for isolation and quota management.

## Related Resources

- [Azure OpenAI](./azure-openai.md) — LLM models (separate from AI Services)
- [Azure AI Document Intelligence](./azure-ai-document-intelligence.md) — Document extraction service (FormRecognizer kind)
- [Azure AI Content Understanding](./azure-ai-content-understanding.md) — Multimodal content analysis (preview)
- [Azure AI Search](./azure-ai-search.md) — Often integrated with Document Intelligence and language models
- [Azure Machine Learning](./azure-machine-learning.md) — Custom model training and deployment
