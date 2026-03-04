# Azure Kubernetes Service (AKS)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.ContainerService` |
| **Resource Type** | `Microsoft.ContainerService/managedClusters` |
| **Azure Portal Category** | Containers > Kubernetes Services |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/aks/intro-kubernetes) |
| **Pricing** | [AKS Pricing](https://azure.microsoft.com/pricing/details/kubernetes-service/) |
| **SLA** | [99.9% (free tier) to 99.99% (Standard/Premium with AZs)](https://azure.microsoft.com/support/legal/sla/kubernetes-service/) |

## Overview

Azure Kubernetes Service (AKS) is a managed Kubernetes cluster service. It abstracts the control plane and provides managed node pools, integrated Entra ID authentication, Azure CNI/Overlay networking, Azure Monitor integration, and Azure RBAC for Kubernetes. In a Workload Landing Zone, AKS runs containerized applications with private cluster API endpoint and workload identity.

## Least-Privilege RBAC Reference

> AKS has **two RBAC layers**: Azure RBAC (cluster infrastructure management) and Kubernetes RBAC (workload access within the cluster). This file covers both.

---

### Azure RBAC — Cluster Infrastructure

#### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create AKS cluster | Resource Group | `Azure Kubernetes Service Contributor Role` | Creates the cluster and the managed `MC_*` node resource group. Also requires `Network Contributor` on the target subnet for Azure CNI. |
| Create node pool | AKS cluster | `Azure Kubernetes Service Contributor Role` | |
| Attach ACR to AKS | AKS + ACR | `Azure Kubernetes Service Contributor Role` + `User Access Administrator` on ACR | Attaching grants `AcrPull` to the kubelet managed identity. |

#### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Scale node pool | AKS cluster | `Azure Kubernetes Service Contributor Role` | |
| Upgrade cluster version | AKS cluster | `Azure Kubernetes Service Contributor Role` | |
| Upgrade node pool OS image | AKS cluster | `Azure Kubernetes Service Contributor Role` | |
| Enable/disable cluster features (OIDC, Workload Identity, KEDA) | AKS cluster | `Azure Kubernetes Service Contributor Role` | |
| Modify network settings | AKS cluster | `Azure Kubernetes Service Contributor Role` | Some network settings cannot be changed after creation. |
| Enable cluster autoscaler | AKS cluster | `Azure Kubernetes Service Contributor Role` | |

#### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete AKS cluster | Resource Group | `Azure Kubernetes Service Contributor Role` | Also deletes the `MC_*` node resource group. |
| Delete a node pool | AKS cluster | `Azure Kubernetes Service Contributor Role` | System node pool cannot be deleted while user node pools exist. |

#### ⚙️ Configure (Infrastructure)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Get cluster admin kubeconfig | AKS cluster | `Azure Kubernetes Service Cluster Admin Role` | Grants full `cluster-admin` Kubernetes RBAC access. Use only for break-glass scenarios. |
| Get cluster user kubeconfig | AKS cluster | `Azure Kubernetes Service Cluster User Role` | Entra ID–authenticated access; Kubernetes RBAC controls what the user can do within the cluster. |
| Configure Diagnostic Settings | AKS cluster | `Monitoring Contributor` | |
| View cluster metrics | AKS cluster | `Reader` | |
| Configure Azure Monitor Container Insights | AKS cluster | `Monitoring Contributor` + `Log Analytics Contributor` | |

---

### Kubernetes RBAC — Cluster Workloads

> Kubernetes RBAC roles are assigned within the cluster using `kubectl` or Azure RBAC (when Azure RBAC for Kubernetes is enabled). The table below covers **Azure RBAC-based Kubernetes roles** (requires `EnableAzureRBAC` feature on cluster).

#### ⚙️ Configure (Workloads)

| Operation | Kubernetes Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read all cluster resources (cluster-wide) | Cluster | `Azure Kubernetes Service RBAC Reader` | Can view pods, deployments, services, etc. Cannot view secrets. |
| Read/write most cluster resources (cluster-wide) | Cluster | `Azure Kubernetes Service RBAC Writer` | Can create/update/delete deployments, pods, configmaps. Cannot manage RBAC. |
| Admin access to a namespace | Namespace | `Azure Kubernetes Service RBAC Admin` | Full access within a namespace including RBAC management. |
| Cluster-wide admin access | Cluster | `Azure Kubernetes Service RBAC Cluster Admin` | Full access to all Kubernetes resources including nodes and CRDs. |

## Role Summary

### Azure Infrastructure Roles

| Role | Create Cluster | Manage Node Pools | Get Admin Kubeconfig | Get User Kubeconfig |
|---|---|---|---|---|
| `Azure Kubernetes Service Contributor Role` | ✅ | ✅ | ❌ | ❌ |
| `Azure Kubernetes Service Cluster Admin Role` | ❌ | ❌ | ✅ | ✅ |
| `Azure Kubernetes Service Cluster User Role` | ❌ | ❌ | ❌ | ✅ |

### Kubernetes RBAC Roles (Azure RBAC mode)

| Role | Read Resources | Write Resources | Manage RBAC | Cluster-wide |
|---|---|---|---|---|
| `Azure Kubernetes Service RBAC Cluster Admin` | ✅ | ✅ | ✅ | ✅ |
| `Azure Kubernetes Service RBAC Admin` | ✅ | ✅ | ✅ | ❌ (namespace) |
| `Azure Kubernetes Service RBAC Writer` | ✅ | ✅ | ❌ | Configurable |
| `Azure Kubernetes Service RBAC Reader` | ✅ | ❌ | ❌ | Configurable |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Container Registry](./azure-container-registry.md) | `Microsoft.ContainerRegistry/registries` | Provides container images for workloads running in AKS; AKS cluster managed identity (or kubelet identity) requires `AcrPull` on the registry. | Optional (required for ACR image pulls) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Secrets Store CSI Driver mounts Key Vault secrets as Kubernetes secrets or volumes; pod workload identity requires `Key Vault Secrets User`. | Optional |
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Persistent Volume Claims backed by Azure Files shares; the AKS managed identity or node pool identity requires `Storage File Data SMB Share Contributor`. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives AKS diagnostic logs (API server, audit, scheduler, controller manager) and Container Insights metrics via Diagnostic Settings and the monitoring add-on. | Optional (strongly recommended) |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides the VNet and subnet for AKS node pools (Azure CNI or Overlay); AKS managed identity requires `Network Contributor` on the subnet. | Required |
| [Azure Monitor](../platform-landing-zone/azure-monitor.md) | `Microsoft.Insights/components` | Provides Container Insights dashboards, alert rules for node/pod health, and Prometheus metrics scraping via the Azure Monitor workspace. | Optional (strongly recommended) |

## Notes / Considerations

- **Workload Identity** (OIDC + Azure Workload Identity) is the recommended way for pods to authenticate to Azure services — replaces Pod Identity (deprecated). Assign Azure RBAC roles to the User-Assigned Managed Identity federated with the Kubernetes service account.
- **Private clusters** (private API server endpoint) require `Network Contributor` on the VNet and a Private DNS Zone for `privatelink.<region>.azmk8s.io`.
- **`Azure Kubernetes Service Cluster Admin Role`** provides `cluster-admin` access — scope to individuals only, not service principals, and use PIM for just-in-time access.
- **System node pool** must run on Linux and cannot be deleted while user node pools exist. Size it appropriately for system workloads (CoreDNS, metrics-server, etc.).
- **Azure CNI Overlay** is recommended over basic Azure CNI for large clusters to conserve IP space; requires `Network Contributor` for pod CIDR configuration.
- **Kubernetes secrets** are not visible to `RBAC Reader` — grant `RBAC Admin` or create a custom ClusterRole if secret access is needed.

## Related Resources

- [Azure Container Registry](./azure-container-registry.md) — Image source for AKS workloads
- [Spoke Virtual Network](./spoke-virtual-network.md) — AKS node and pod networking
- [Azure Key Vault](./azure-key-vault.md) — Secrets via CSI Secret Store driver
- [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) — Private cluster DNS
