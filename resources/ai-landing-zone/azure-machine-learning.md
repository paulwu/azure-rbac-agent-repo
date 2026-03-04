# Azure Machine Learning

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.MachineLearningServices` |
| **Resource Types** | `Microsoft.MachineLearningServices/workspaces`, `Microsoft.MachineLearningServices/workspaces/computes`, `Microsoft.MachineLearningServices/workspaces/datastores` |
| **Azure Portal Category** | AI + Machine Learning > Machine Learning |
| **Landing Zone Context** | AI Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/machine-learning/overview-what-is-azure-machine-learning) |
| **Pricing** | [AML Pricing](https://azure.microsoft.com/pricing/details/machine-learning/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/machine-learning/) |

## Overview

Azure Machine Learning (AML) is a platform for building, training, deploying, and managing machine learning models. In an AI Landing Zone it provides the central workspace for data scientists, ML engineers, and MLOps teams. AML integrates with ADLS Gen2, Key Vault, Container Registry, and Azure Monitor.

## Least-Privilege RBAC Reference

> AML has purpose-built roles for data scientists and compute operators. Always prefer these over `Contributor` for ML workload identities.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create AML Workspace | Resource Group | `Contributor` | Creates workspace and linked resources (Key Vault, Storage, ACR, App Insights). |
| Create Compute Instance | Workspace | `AzureML Data Scientist` | Compute Instance is a personal, managed dev VM. |
| Create Compute Cluster | Workspace | `AzureML Compute Operator` | Cluster for training jobs. |
| Create Inference Endpoint (Managed Online/Batch) | Workspace | `AzureML Data Scientist` | Deploys models as REST endpoints. |
| Create Datastore | Workspace | `AzureML Data Scientist` | Links to ADLS Gen2 / Blob for data access. |
| Create Environment | Workspace | `AzureML Data Scientist` | Docker environments for training/inference. |
| Register a Model | Workspace | `AzureML Data Scientist` | Registers model artifacts in the model registry. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Submit a training job / experiment | Workspace | `AzureML Data Scientist` | |
| Edit a pipeline component | Workspace | `AzureML Data Scientist` | |
| Update a deployed endpoint | Workspace | `AzureML Data Scientist` | |
| Modify compute cluster settings | Workspace | `AzureML Compute Operator` | Scale cluster, change VM size. |
| Start/stop Compute Instance | Workspace | `AzureML Data Scientist` (own) / `AzureML Compute Operator` (others') | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a compute resource | Workspace | `AzureML Compute Operator` | |
| Delete a model | Workspace | `AzureML Data Scientist` | |
| Delete an endpoint | Workspace | `AzureML Data Scientist` | |
| Delete AML Workspace | Resource Group | `Contributor` | Also removes linked resources unless detached. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read workspace (view experiments, models, datasets) | Workspace | `Reader` | View-only access to workspace metadata. |
| Read workspace secrets (connection strings) | Workspace | `AzureML Workspace Connection Secrets Reader` | Read workspace connection secrets for linked services. |
| Configure workspace networking (Private Endpoint, VNet) | Workspace | `Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Workspace | `Monitoring Contributor` | |
| Configure CMK for workspace | Workspace + Key Vault | `Contributor` + `Key Vault Crypto Service Encryption User` | |
| Assign workspace roles | Workspace | `Owner` or `User Access Administrator` | |

## AML Role Summary

| Role | Create Compute | Submit Jobs | Deploy Models | Register Models | Manage Workspace |
|---|---|---|---|---|---|
| `AzureML Data Scientist` | ✅ (Compute Instance) | ✅ | ✅ | ✅ | ❌ |
| `AzureML Compute Operator` | ✅ (Clusters) | ❌ | ❌ | ❌ | ❌ |
| `AzureML Registry User` | ❌ | ❌ | ❌ | ✅ (Registry) | ❌ |
| `Reader` | ❌ | ❌ | ❌ | ❌ | ❌ (view only) |

## AML Managed Identity — Required Roles on Connected Resources

| Resource | Role | Purpose |
|---|---|---|
| Storage Account (default) | `Storage Blob Data Contributor` | Read/write training data and outputs |
| Container Registry (default ACR) | `AcrPull` | Pull training environment images |
| Key Vault (default) | `Key Vault Secrets User` | Read workspace secrets |
| ADLS Gen2 Datastore | `Storage Blob Data Contributor` | Data access for training jobs |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Default datastore for training data, model artifacts, and experiment outputs; the workspace managed identity requires `Storage Blob Data Contributor`. | Required |
| [Azure Container Registry](../workload-landing-zone/azure-container-registry.md) | `Microsoft.ContainerRegistry/registries` | Stores Docker training and inference environment images; workspace managed identity requires `AcrPull` to pull images for compute runs. | Required |
| [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores workspace secrets (datastore connection strings, service credentials); workspace managed identity requires `Key Vault Secrets User`. | Required |
| [Azure Data Lake Storage Gen2](../data-landing-zone/azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | External datastore for large-scale training datasets; job managed identity or workspace managed identity requires `Storage Blob Data Contributor`. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Application Insights training telemetry and workspace diagnostic logs via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides managed VNet for compute clusters and Private Endpoint connectivity for all workspace linked resources in a private workspace configuration. | Optional (strongly recommended) |

## Notes / Considerations

- **`AzureML Data Scientist`** is the recommended role for data scientists — it provides broad workspace access for experimentation without management-plane control.
- **`AzureML Compute Operator`** is for platform/MLOps teams managing shared compute — it cannot run experiments.
- **Compute Instance** should be assigned a **managed identity** for accessing data — avoids credential sharing between team members.
- **Private workspace** (all resources on private endpoints) requires significant networking configuration in the AI Landing Zone — plan DNS and Private Endpoint deployment carefully.
- **Managed Online Endpoints** use blue/green deployment with traffic splitting — managed identity on the endpoint needs `AcrPull` and storage read access.
- Use **Azure Machine Learning Registries** for sharing models and components across workspaces/environments — access via `AzureML Registry User`.

## Related Resources

- [Azure OpenAI](./azure-openai.md) — Foundation model fine-tuning via AML
- [Azure Container Registry](../workload-landing-zone/azure-container-registry.md) — Training environment images
- [Azure Data Lake Storage Gen2](../data-landing-zone/azure-data-lake-storage-gen2.md) — Training data
- [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) — Workspace secrets
