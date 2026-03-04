# Azure Bastion

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Type** | `Microsoft.Network/bastionHosts` |
| **Azure Portal Category** | Networking > Bastions |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/bastion/bastion-overview) |
| **Pricing** | [Azure Bastion Pricing](https://azure.microsoft.com/pricing/details/azure-bastion/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/azure-bastion/) |

## Overview

Azure Bastion provides secure, browser-based RDP/SSH access to VMs without exposing public IPs or opening inbound ports. In a Platform Landing Zone, a single Bastion host deployed in the hub VNet can be shared with spoke VNets via VNet peering (Premium SKU) or deployed per-spoke (Basic/Standard SKU).

## Least-Privilege RBAC Reference

> Bastion access requires **two separate roles**: one to manage the Bastion resource itself (`Network Contributor`) and another to actually connect to a target VM.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Azure Bastion | Resource Group | `Network Contributor` | Requires a pre-existing subnet named `AzureBastionSubnet` (minimum /26). Also requires a Standard SKU public IP. |
| Create `AzureBastionSubnet` | VNet | `Network Contributor` | Subnet must be /26 or larger. No NSG rules should block Bastion management ports (443, 4443). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Upgrade Bastion SKU (Basic → Standard → Premium) | Bastion resource | `Network Contributor` | In-place upgrade is supported for some SKU transitions. |
| Enable/disable features (file copy, IP-based connection, shareable links) | Bastion resource | `Network Contributor` | Feature availability depends on SKU. |
| Scale instance count (Standard SKU) | Bastion resource | `Network Contributor` | Autoscaling is based on concurrent sessions. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Azure Bastion | Resource Group | `Network Contributor` | Removing Bastion does not affect the VMs or VNet. |

### ⚙️ Configure — Bastion Management

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings | Bastion resource | `Monitoring Contributor` | Captures session audit logs (who connected to which VM). |
| Configure NSG rules for AzureBastionSubnet | NSG resource | `Network Contributor` | Required inbound/outbound rules must not be blocked. |

### ⚙️ Configure — End User VM Access

> The roles below are for **end users** who need to connect to VMs through Bastion. These are separate from managing the Bastion resource itself.

| Connection Type | VM Scope | Required Role(s) | Notes |
|---|---|---|---|
| RDP / SSH to Windows/Linux VM (password or SSH key) | Target VM | `Virtual Machine User Login` (non-admin) or `Virtual Machine Administrator Login` (admin) | User also needs **Reader** on the Bastion host resource. Without Reader on Bastion, the Connect blade will not appear. |
| Connect to VM using native client (Bastion Standard) | Target VM | `Virtual Machine User Login` or `Virtual Machine Administrator Login` | Same as above; native client tunnels through Bastion. |
| Connect using shareable link | Target VM | No Azure RBAC required | Shareable links bypass RBAC — use with caution and time-limit exposure. |
| File copy (upload/download) | Target VM | `Virtual Machine User Login` or `Virtual Machine Administrator Login` | Bastion Standard SKU feature. |

## Minimum End-User Role Assignment Summary

| Role | Scope | Purpose |
|---|---|---|
| `Reader` | Bastion Host resource | Required to see and use the Connect blade in Azure Portal |
| `Virtual Machine User Login` | Target VM resource | Non-privileged login (standard user account) |
| `Virtual Machine Administrator Login` | Target VM resource | Privileged login (local admin / root) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Azure Bastion is deployed into the `AzureBastionSubnet` of the hub VNet; the subnet must be at least /27. | Required |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Bastion diagnostic logs (session audit events, connection logs) via Diagnostic Settings for access auditing. | Optional (strongly recommended) |

## Notes / Considerations

- **`Virtual Machine User Login`** and **`Virtual Machine Administrator Login`** use Azure AD (Entra ID) authentication — the VM must have the **Azure AD Login** extension installed (`AADLoginForWindows` / `AADSSHLoginForLinux`).
- For VMs using **local account authentication**, the user needs `Reader` on Bastion and `Reader` on the VM (to find it), plus knowledge of the local credentials — no Entra ID RBAC role grants the password.
- **Premium SKU** enables IP-based connections and cross-VNet connectivity without requiring VNet peering to the hub.
- **Audit logging** via Diagnostic Settings is strongly recommended in Platform LZs to maintain an access audit trail.
- Do **not** deploy additional VMs or resources in `AzureBastionSubnet` — it is exclusively for the Bastion host.

## Related Resources

- [Hub Virtual Network](./hub-virtual-network.md) — Bastion deployed in hub's AzureBastionSubnet
- [Virtual Machines](../workload-landing-zone/virtual-machines.md) — Target resources for Bastion access
