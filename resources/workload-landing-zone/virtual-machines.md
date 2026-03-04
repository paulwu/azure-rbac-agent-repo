# Azure Virtual Machines

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Compute` |
| **Resource Types** | `Microsoft.Compute/virtualMachines`, `Microsoft.Compute/disks`, `Microsoft.Compute/snapshots`, `Microsoft.Compute/virtualMachineScaleSets` |
| **Azure Portal Category** | Compute > Virtual Machines |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/virtual-machines/) |
| **Pricing** | [VM Pricing](https://azure.microsoft.com/pricing/details/virtual-machines/) |
| **SLA** | [99.9% (single VM Premium SSD) to 99.99% (Availability Zones)](https://azure.microsoft.com/support/legal/sla/virtual-machines/) |

## Overview

Azure Virtual Machines provide IaaS compute in the workload landing zone. VMs are used for lift-and-shift workloads, legacy applications, and scenarios where PaaS is not suitable. VM management involves the compute resource, OS disks, NICs, extensions, and optionally Azure AD (Entra ID) login integration.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Virtual Machine | Resource Group | `Virtual Machine Contributor` | Creates the VM, OS disk, and NIC. Also requires `Network Contributor` on the target subnet to attach the NIC, unless the NIC is pre-created. |
| Create a Managed Disk | Resource Group | `Virtual Machine Contributor` | Includes `Microsoft.Compute/disks/write`. |
| Create a VM Scale Set (VMSS) | Resource Group | `Virtual Machine Contributor` | |
| Create a VM from Shared Image Gallery | Image resource | `Reader` on the image + `Virtual Machine Contributor` on target RG | |
| Assign a User-Assigned Managed Identity to VM | VM resource | `Virtual Machine Contributor` + `Managed Identity Operator` | `Managed Identity Operator` is required on the managed identity resource. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Resize a VM (change SKU) | VM resource | `Virtual Machine Contributor` | Requires VM stop/deallocate in most cases. |
| Add/resize/detach managed disks | VM + Disk resource | `Virtual Machine Contributor` | |
| Modify VM extensions | VM resource | `Virtual Machine Contributor` | |
| Update VM tags | VM resource | `Tag Contributor` | Minimum role for tag-only changes. |
| Modify boot diagnostics | VM resource | `Virtual Machine Contributor` + `Storage Blob Data Contributor` | Boot diagnostics writes screenshots to a Storage Account. |
| Enable Azure Monitor Agent (AMA) | VM resource | `Virtual Machine Contributor` | AMA is deployed as a VM extension. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a VM | Resource Group | `Virtual Machine Contributor` | Does NOT automatically delete associated disks, NICs, or public IPs unless explicitly selected. |
| Delete a Managed Disk | Resource Group | `Virtual Machine Contributor` | Disk must be detached first. |
| Delete a Snapshot | Resource Group | `Virtual Machine Contributor` | |

### ⚙️ Configure — Operations

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Start / Stop / Restart VM | VM resource | `Virtual Machine Contributor` | `Virtual Machine Operator` (preview) provides start/stop without contributor access. |
| Deallocate VM | VM resource | `Virtual Machine Contributor` | |
| Capture VM image | VM resource | `Virtual Machine Contributor` | |
| Run Command on VM | VM resource | `Virtual Machine Contributor` | Executes scripts inside the VM via the Azure Agent. |
| Reset VM password | VM resource | `Virtual Machine Contributor` | |
| Configure Disk Encryption (ADE) | VM + Key Vault | `Virtual Machine Contributor` + `Key Vault Crypto Officer` | Azure Disk Encryption uses Key Vault for key storage. |
| Enable CMK for managed disk | Disk + Key Vault | `Virtual Machine Contributor` + `Key Vault Crypto Service Encryption User` | Disk Encryption Set (`Microsoft.Compute/diskEncryptionSets`) is an intermediary resource. |
| Configure Diagnostic Settings | VM resource | `Monitoring Contributor` | |
| Enable JIT VM Access (request) | VM resource | Custom role with `Microsoft.Security/locations/jitNetworkAccessPolicies/initiate/action` | See [Microsoft Defender for Cloud](../platform-landing-zone/microsoft-defender-for-cloud.md). |

### ⚙️ Configure — Login (Entra ID Authentication)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Login as local administrator / root | VM resource | `Virtual Machine Administrator Login` | Entra ID-joined VM with `AADLoginForWindows` / `AADSSHLoginForLinux` extension. |
| Login as standard user | VM resource | `Virtual Machine User Login` | Non-privileged OS login. |

> **Note**: `Virtual Machine Administrator Login` and `Virtual Machine User Login` only apply to VMs with Entra ID login extension installed. For VMs using local credentials, these roles have no effect — the user needs the password/key.

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides the subnet to which the VM's Network Interface Card (NIC) is attached; a VM cannot be created without a target virtual network and subnet. | Required |
| [Network Security Groups](./network-security-groups.md) | `Microsoft.Network/networkSecurityGroups` | Controls inbound and outbound traffic to the VM's NIC or subnet; strongly recommended to restrict access. | Optional (strongly recommended) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores disk encryption keys (Azure Disk Encryption or Customer-Managed Keys via Disk Encryption Set) and SSH private keys; required when enabling disk encryption at rest. | Required (disk encryption) / Optional |
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Used for boot diagnostics screenshot storage when boot diagnostics is enabled on the VM. | Optional |
| [Recovery Services Vault](../platform-landing-zone/recovery-services-vault.md) | `Microsoft.RecoveryServices/vaults` | Manages VM backup policies and recovery points when Azure Backup is enabled for the VM. | Optional |

## Notes / Considerations

- **`Virtual Machine Contributor`** does NOT grant login access to the VM operating system — OS access is controlled separately by Entra ID login roles or local credentials.
- **`Virtual Machine Operator`** (preview role as of 2024) provides start/stop/restart without full contributor rights — check for GA status in your environment.
- **System-assigned Managed Identity** on a VM requires no special role to enable during creation; `Virtual Machine Contributor` covers it. The identity then needs roles on any resources it accesses.
- For **VMSS (Autoscale)**, the autoscale settings resource (`Microsoft.Insights/autoscaleSettings`) requires `Monitoring Contributor`.
- **Azure Backup** for VMs requires `Backup Contributor` on the Recovery Services Vault and `Virtual Machine Contributor` on the VM.
- **Azure Site Recovery** requires `Site Recovery Contributor` role on both the vault and the VMs.

## Related Resources

- [Azure SSH Key](./azure-ssh-key.md) — SSH public key resources for Linux VM provisioning
- [Network Security Groups](./network-security-groups.md) — Applied to the VM's NIC and subnet
- [Azure Bastion](../platform-landing-zone/azure-bastion.md) — Secure VM access without public IPs
- [Azure Key Vault](./azure-key-vault.md) — Disk encryption key storage
- [Recovery Services Vault](../platform-landing-zone/recovery-services-vault.md) — VM backup and Site Recovery
