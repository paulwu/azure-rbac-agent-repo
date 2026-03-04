# Azure Firewall

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Types** | `Microsoft.Network/azureFirewalls`, `Microsoft.Network/firewallPolicies`, `Microsoft.Network/firewallPolicies/ruleCollectionGroups` |
| **Azure Portal Category** | Networking > Azure Firewall |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/firewall/overview) |
| **Pricing** | [Azure Firewall Pricing](https://azure.microsoft.com/pricing/details/azure-firewall/) |
| **SLA** | [99.99%](https://azure.microsoft.com/support/legal/sla/azure-firewall/) |

## Overview

Azure Firewall is a managed, cloud-native network security service deployed in the hub VNet. It provides centralized inbound/outbound traffic inspection and filtering for all spoke VNets in the landing zone. Azure Firewall Premium adds IDPS, TLS inspection, and URL filtering.

## Least-Privilege RBAC Reference

> Azure Firewall management uses the `Network Contributor` role. There is no purpose-built least-privilege role specific to Azure Firewall — `Network Contributor` is the minimum that covers all firewall operations.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Azure Firewall instance | Resource Group | `Network Contributor` | Requires a pre-existing subnet named `AzureFirewallSubnet` (minimum /26). Also requires a Public IP (`Microsoft.Network/publicIPAddresses`). |
| Create Firewall Policy | Resource Group | `Network Contributor` | Firewall Policy is a separate resource from the firewall instance. |
| Create Rule Collection Group | Firewall Policy | `Network Contributor` | Rule Collection Groups contain Network, Application, and DNAT rules. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Add/modify network rules | Firewall Policy | `Network Contributor` | Changes take effect within 1–2 minutes. |
| Add/modify application rules | Firewall Policy | `Network Contributor` | |
| Add/modify DNAT rules | Firewall Policy | `Network Contributor` | |
| Change firewall SKU (Standard → Premium) | Firewall resource | `Network Contributor` | SKU upgrade is non-disruptive but irreversible. |
| Update threat intelligence settings | Firewall Policy | `Network Contributor` | |
| Configure DNS proxy settings | Firewall Policy | `Network Contributor` | Required for FQDN-based filtering in network rules. |
| Associate/dissociate Firewall Policy | Firewall resource | `Network Contributor` | |
| Enable forced tunneling | Firewall resource | `Network Contributor` | Requires a management subnet (`AzureFirewallManagementSubnet`). |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Azure Firewall | Resource Group | `Network Contributor` | All route tables pointing to the firewall should be updated first. |
| Delete Firewall Policy | Resource Group | `Network Contributor` | Must be disassociated from all firewalls before deletion. |
| Delete Rule Collection Group | Firewall Policy | `Network Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings (firewall logs) | Firewall resource | `Monitoring Contributor` | Sends Azure Firewall logs and metrics to Log Analytics / Event Hub / Storage. |
| Configure Firewall Workbooks | Log Analytics Workspace | `Log Analytics Contributor` | Azure Firewall Workbook requires workspace contributor for setup. |
| Configure Structured Logging | Firewall resource / Policy | `Network Contributor` | Structured logging mode is recommended for new deployments. |
| Enable IDPS (Premium SKU) | Firewall Policy | `Network Contributor` | Requires Premium SKU on both firewall and policy. |
| Configure TLS Inspection (Premium) | Firewall Policy + Key Vault | `Network Contributor` + `Key Vault Certificates Officer` | TLS inspection certificate is stored in Key Vault; firewall uses a managed identity to read it. |
| Assign managed identity to Firewall | Firewall resource | `Network Contributor` + `Managed Identity Operator` | Required for TLS inspection Key Vault access. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Azure Firewall is deployed into the `AzureFirewallSubnet` of the hub VNet; the VNet provides the network fabric for traffic routing. | Required |
| [Azure Firewall Policy](./hub-virtual-network.md) | `Microsoft.Network/firewallPolicies` | Firewall Policy resource manages DNAT, network, and application rule collections; `Network Contributor` on the policy is required to update rules. | Required |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Azure Firewall diagnostic logs (application rules, network rules, DNS proxy, IDPS alerts) via Diagnostic Settings for traffic auditing. | Required (strongly recommended) |
| [Azure Monitor](./azure-monitor.md) | `Microsoft.Insights/components` | Provides alert rules on Firewall threat intelligence events and traffic anomalies. | Optional |

## Notes / Considerations

- **`Network Contributor`** is broadly scoped — restrict assignments to the specific resource group containing the firewall and its policy.
- **Azure Firewall Manager** provides a centralized UI for managing firewall policies across multiple firewalls; it uses the same `Network Contributor` role.
- **Firewall Policy hierarchy**: Parent policies define base rules; child policies inherit and extend them. Rule priority ordering is managed within each policy independently.
- **Public IP SKU**: Azure Firewall requires a **Standard SKU** public IP. Basic SKU is not supported.
- **Availability Zones**: Deploy Azure Firewall across all available AZs for maximum resilience; this is set at creation time and cannot be changed.
- **Forced tunneling** to an on-premises NVA requires additional configuration of a management public IP and subnet.

## Related Resources

- [Hub Virtual Network](./hub-virtual-network.md) — Firewall is deployed in the hub
- [VPN / ExpressRoute Gateway](./vpn-expressroute-gateway.md) — On-premises traffic routed through firewall
- [Log Analytics Workspace](./log-analytics-workspace.md) — Firewall diagnostic logs destination
