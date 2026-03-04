# Private DNS Zones

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Types** | `Microsoft.Network/privateDnsZones`, `Microsoft.Network/privateDnsZones/virtualNetworkLinks`, `Microsoft.Network/privateDnsZones/A` (and other record types) |
| **Azure Portal Category** | Networking > Private DNS Zones |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/dns/private-dns-overview) |
| **Pricing** | [Azure DNS Pricing](https://azure.microsoft.com/pricing/details/dns/) |
| **SLA** | [99.99%](https://azure.microsoft.com/support/legal/sla/azure-dns/) |

## Overview

Private DNS Zones provide name resolution for Azure Private Endpoints and internal services without requiring custom DNS servers. In a Platform Landing Zone, Private DNS Zones are centrally hosted in the hub subscription and linked to all VNets (hub and spokes) for unified private endpoint resolution. Common zones include `privatelink.blob.core.windows.net`, `privatelink.vaultcore.azure.net`, etc.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Private DNS Zone | Resource Group | `Private DNS Zone Contributor` | Creates the zone resource (e.g., `privatelink.blob.core.windows.net`). |
| Create a VNet Link (link zone to VNet) | Private DNS Zone resource | `Private DNS Zone Contributor` | Also requires `Reader` or `Network Contributor` on the target VNet to validate the link. For auto-registration, `Network Contributor` on the VNet is needed. |
| Create a DNS record set (A, CNAME, etc.) | Private DNS Zone resource | `Private DNS Zone Contributor` | A records for private endpoints are typically auto-created by the private endpoint provisioning process. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify a DNS record (IP address, TTL) | Private DNS Zone resource | `Private DNS Zone Contributor` | |
| Update VNet Link settings (enable/disable auto-registration) | VNet Link resource | `Private DNS Zone Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a DNS record set | Private DNS Zone resource | `Private DNS Zone Contributor` | |
| Delete a VNet Link | Private DNS Zone resource | `Private DNS Zone Contributor` | |
| Delete a Private DNS Zone | Resource Group | `Private DNS Zone Contributor` | All record sets and VNet links must be removed first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Auto-create DNS record via Private Endpoint | Private DNS Zone resource | `Private DNS Zone Contributor` | When creating a private endpoint and selecting the auto-DNS-registration option, the provisioning identity needs `Private DNS Zone Contributor` on the target zone. |
| Read DNS records | Private DNS Zone resource | `Reader` | View records and zone configuration. |
| Configure DNS Private Resolver (inbound/outbound endpoints) | Resource Group | `Network Contributor` | DNS Private Resolver (`Microsoft.Network/dnsResolvers`) is a separate resource. |

## Common Private DNS Zone Names

| Azure Service | Private DNS Zone |
|---|---|
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure File Storage | `privatelink.file.core.windows.net` |
| Azure Queue Storage | `privatelink.queue.core.windows.net` |
| Azure Table Storage | `privatelink.table.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |
| Azure SQL Database | `privatelink.database.windows.net` |
| Azure Cosmos DB (SQL) | `privatelink.documents.azure.com` |
| Azure Event Hubs | `privatelink.servicebus.windows.net` |
| Azure Container Registry | `privatelink.azurecr.io` |
| Azure Kubernetes Service API | `privatelink.<region>.azmk8s.io` |
| Azure Machine Learning | `privatelink.api.azureml.ms` |
| Azure OpenAI | `privatelink.openai.azure.com` |
| Azure Monitor / Log Analytics | `privatelink.monitor.azure.com`, `privatelink.ods.opinsights.azure.com` |
| Azure Data Factory | `privatelink.datafactory.azure.net` |
| Azure Synapse Analytics | `privatelink.sql.azuresynapse.net`, `privatelink.dev.azuresynapse.net` |

> For a full list, see [Azure Private Endpoint DNS configuration](https://learn.microsoft.com/azure/private-link/private-endpoint-dns).

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Private DNS zones must be linked to the hub VNet (and/or spoke VNets) via Virtual Network Links so that DNS resolution is available to all connected resources. | Required |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Private DNS Zone query logs via Diagnostic Settings for DNS traffic monitoring and troubleshooting. | Optional |

## Notes / Considerations

- **`Private DNS Zone Contributor`** is narrower than `Network Contributor` — prefer it for DNS-only operations.
- **Auto-registration** (automatic DNS record creation when VMs join a VNet) works only with standard (non-privatelink) private DNS zones; privatelink zones require explicit A record creation.
- In a Platform LZ, use **Azure Policy** (`DeployIfNotExists`) to automatically create DNS records when private endpoints are provisioned in spoke subscriptions — the policy managed identity needs `Private DNS Zone Contributor` on the centralized zones.
- **DNS Private Resolver** (recommended over custom DNS VMs) forwards conditional queries to on-premises DNS; deploying it requires `Network Contributor` in addition to DNS zone management roles.
- All spoke VNets should be linked to platform-managed Private DNS Zones with **auto-registration disabled** (to prevent spoke VMs from polluting the zone with auto-registered names).

## Related Resources

- [Hub Virtual Network](./hub-virtual-network.md) — Zones are linked to the hub VNet
- [Azure Key Vault](./azure-key-vault.md) — Uses `privatelink.vaultcore.azure.net`
- [Azure Bastion](./azure-bastion.md) — Bastion itself does not require a private DNS zone
