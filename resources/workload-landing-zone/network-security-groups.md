# Network Security Groups (NSG)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Type** | `Microsoft.Network/networkSecurityGroups`, `Microsoft.Network/networkSecurityGroups/securityRules` |
| **Azure Portal Category** | Networking > Network Security Groups |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/virtual-network/network-security-groups-overview) |
| **Pricing** | No charge for NSG; [flow logs charged by data volume](https://azure.microsoft.com/pricing/details/network-watcher/) |
| **SLA** | Covered by VNet SLA — [99.9%](https://azure.microsoft.com/support/legal/sla/virtual-network/) |

## Overview

Network Security Groups (NSGs) are stateful L3/L4 packet filters applied to subnets or individual NICs. They control inbound and outbound traffic using priority-ordered security rules. In a Workload Landing Zone, NSGs are the primary micro-segmentation mechanism, typically applied at subnet level. NSG Flow Logs provide traffic visibility for security analysis.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create an NSG | Resource Group | `Network Contributor` | |
| Add a security rule to an NSG | NSG resource | `Network Contributor` | |
| Associate NSG with a subnet | VNet / Subnet | `Network Contributor` on the VNet | |
| Associate NSG with a NIC | NIC resource | `Network Contributor` | |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify a security rule (priority, ports, action) | NSG resource | `Network Contributor` | Changes take effect within ~30 seconds. |
| Change rule priority | NSG resource | `Network Contributor` | |
| Reassociate NSG to a different subnet/NIC | NSG + target resource | `Network Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete an NSG rule | NSG resource | `Network Contributor` | |
| Disassociate NSG from subnet/NIC | Subnet or NIC | `Network Contributor` | |
| Delete an NSG | Resource Group | `Network Contributor` | Must be disassociated from all subnets and NICs first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Enable NSG Flow Logs | NSG resource | `Network Contributor` + `Storage Blob Data Contributor` | Flow logs are written to a Storage Account; enabling requires contributor on both the NSG and the storage account. |
| Enable Traffic Analytics (from flow logs) | NSG + Log Analytics Workspace | `Network Contributor` + `Log Analytics Contributor` | Traffic Analytics processes flow logs into the workspace. |
| View effective security rules | VM NIC resource | `Reader` on NIC and NSG | Read-only view of all active rules evaluated for a specific NIC. |
| Read NSG configuration | NSG resource | `Reader` | View-only access to rules and associations. |
| Configure Diagnostic Settings | NSG resource | `Monitoring Contributor` | Sends NSG event and rule counter logs to Log Analytics / Storage. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | NSGs are associated with subnets or NICs within the spoke VNet; `Network Contributor` on the subnet is required to associate or disassociate an NSG. | Required |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives NSG Flow Logs via Network Watcher for traffic analysis and security investigation. | Optional (strongly recommended) |
| [Azure Monitor](../platform-landing-zone/azure-monitor.md) | `Microsoft.Insights/components` | Provides alert rules on NSG security event metrics and flow log anomalies. | Optional |

## Notes / Considerations

- **`Network Contributor`** is the minimum for all NSG operations — there is no purpose-built NSG-specific role.
- **NSG Flow Logs v2** (recommended over v1) are written to Storage Account in JSON format. Use **Traffic Analytics** in Azure Monitor for visualization.
- Apply NSGs at **subnet level** (not NIC level) for consistent policy — NIC-level NSGs are evaluated in addition to subnet-level NSGs, with subnet rules evaluated first for inbound and NIC rules first for outbound.
- **Default deny rules** exist at priority 65000–65500 — never delete the default deny-all rule unless you have explicit allow rules above it.
- **Application Security Groups (ASGs)** (`Microsoft.Network/applicationSecurityGroups`) allow rule definition by logical group rather than IP — same `Network Contributor` role covers ASG management.
- For **Azure Service Tags** in rules (e.g., `AzureLoadBalancer`, `VirtualNetwork`), no special permissions are needed — service tags are Azure-managed.

## Related Resources

- [Spoke Virtual Network](./spoke-virtual-network.md) — NSGs are associated with spoke subnets
- [Virtual Machines](./virtual-machines.md) — NSG also applied at NIC level
- [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) — Destination for flow logs via Traffic Analytics
