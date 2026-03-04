# Spoke Virtual Network

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Type** | `Microsoft.Network/virtualNetworks`, `Microsoft.Network/virtualNetworkPeerings` |
| **Azure Portal Category** | Networking > Virtual Networks |
| **Landing Zone Context** | Workload Landing Zone (Spoke) |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/virtual-network/virtual-networks-overview) |
| **Pricing** | [VNet Pricing](https://azure.microsoft.com/pricing/details/virtual-network/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/virtual-network/) |

## Overview

A Spoke Virtual Network is the network boundary for a workload subscription. It connects to the Platform hub VNet via VNet peering for access to shared services (Firewall, Gateway, DNS). Workload teams manage their spoke VNet independently while the platform team manages the hub-side of the peering.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Spoke VNet | Resource Group | `Network Contributor` | |
| Create subnets | Spoke VNet | `Network Contributor` | |
| Initiate peering to hub (spoke side) | Spoke VNet | `Network Contributor` | Platform team must accept/create the hub-side peering with `Network Contributor` on the hub VNet. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Add/modify subnets | Spoke VNet | `Network Contributor` | |
| Modify address space | Spoke VNet | `Network Contributor` | Cannot overlap with hub or other spokes. |
| Modify DNS settings | Spoke VNet | `Network Contributor` | Custom DNS should point to hub DNS Private Resolver or Azure DNS. |
| Update UDR on subnet (force traffic to firewall) | Route Table | `Network Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Spoke VNet | Resource Group | `Network Contributor` | All resources and peerings must be removed first. |
| Remove VNet Peering (spoke side) | Spoke VNet | `Network Contributor` | Coordinate with platform team to clean up hub side. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Assign NSG to subnet | Subnet | `Network Contributor` | |
| Assign Route Table to subnet | Subnet | `Network Contributor` | |
| Enable service endpoints | Subnet | `Network Contributor` | |
| Configure Private Endpoints in subnet | Subnet + target resource | `Network Contributor` + resource-specific contributor | Private endpoint creation also requires `Microsoft.Network/privateEndpoints/write`. |
| Configure Diagnostic Settings (flow logs) | VNet resource | `Monitoring Contributor` | |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Hub Virtual Network](../platform-landing-zone/hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides hub-spoke VNet peering for centralized routing, firewall inspection, and shared services access; `Network Contributor` on both hub and spoke is required to establish peering. | Required (hub-spoke topology) |
| [Azure Firewall](../platform-landing-zone/azure-firewall.md) | `Microsoft.Network/azureFirewalls` | Acts as the next-hop for spoke traffic via a User-Defined Route; the spoke route table must point to the Azure Firewall private IP for centralized traffic inspection. | Optional (required for hub-spoke inspection) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Spoke VNet is linked to hub-hosted private DNS zones for Private Endpoint DNS resolution; `Private DNS Zone Contributor` required to create VNet links. | Required (for Private Endpoint DNS resolution) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives VNet diagnostic logs and NSG Flow Logs via Network Watcher and Diagnostic Settings. | Optional |

## Notes / Considerations

- **Hub-side peering** must be created by the platform team — workload teams cannot directly create the hub-side peering without access to the platform subscription.
- Use **Azure Policy** with `DeployIfNotExists` to auto-peer new spoke VNets to the hub and configure UDRs.
- Spoke VNet address space must be pre-allocated from a centrally managed IP address management (IPAM) plan to avoid overlaps.
- Never route internet traffic directly from a spoke — all egress should traverse the hub firewall via a default UDR (`0.0.0.0/0 → Firewall IP`).

## Related Resources

- [Hub Virtual Network](../platform-landing-zone/hub-virtual-network.md) — Hub side of the peering
- [Network Security Groups](./network-security-groups.md) — Applied to spoke subnets
