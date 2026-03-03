# Azure RBAC Least-Privilege Reference Library

A structured reference library of Azure Role-Based Access Control (RBAC) least-privilege role assignments, organized by Azure Landing Zone archetype. The repository also includes a **GitHub Copilot custom agent** — the Azure RBAC Advisor — that lets you query this library conversationally.

---

## Purpose

This repository helps platform engineers, security teams, and workload developers answer the question:

> *"What is the minimum Azure RBAC role required to perform this operation on this resource?"*

Every resource file documents the least-privileged built-in role needed for **Create**, **Edit**, **Delete**, and **Configure** operations, with explicit separation of **management plane** and **data plane** roles where applicable (e.g., Azure Storage, Key Vault, Cosmos DB).

---

## Repository Structure

```
.
├── resources/
│   ├── README.md                        # Library overview and RBAC principles
│   ├── platform-landing-zone/           # 12 resources (shared services, governance)
│   ├── workload-landing-zone/           # 11 resources (application workloads)
│   ├── data-landing-zone/               # 9 resources (data platforms and analytics)
│   └── ai-landing-zone/                 # 7 resources (AI/ML services)
├── .github/
│   ├── agents/
│   │   └── azure-rbac-advisor.agent.md  # Copilot custom agent
│   └── copilot-instructions.md          # Authoring rules for future contributions
└── README.md
```

### Resources by Landing Zone

| Landing Zone | Resources Covered |
|---|---|
| **Platform** | Management Groups, Azure Policy, Log Analytics Workspace, Azure Monitor, Microsoft Defender for Cloud, Azure Key Vault, Hub Virtual Network, Azure Firewall, VPN/ExpressRoute Gateway, Azure Bastion, Private DNS Zones, Azure Automation Account |
| **Workload** | Spoke Virtual Network, Network Security Groups, Virtual Machines, App Service, Azure SQL Database, Azure Storage Account *(with Blob/File/Queue/Table breakdowns)*, Azure Key Vault, Azure Load Balancer, Application Gateway, Azure Container Registry, Azure Kubernetes Service |
| **Data** | Azure Data Factory, Azure Synapse Analytics, Azure Data Lake Storage Gen2, Azure Databricks, Azure Event Hubs, Azure Cosmos DB, Azure Stream Analytics, Microsoft Purview, Azure Data Explorer |
| **AI** | Azure Machine Learning, Azure OpenAI, Azure AI Services, Azure AI Search, Azure Bot Service, Azure Applied AI Services, Azure AI Foundry |

---

## Using the Azure RBAC Advisor (Copilot Custom Agent)

The **Azure RBAC Advisor** is a GitHub Copilot custom agent defined in `.github/agents/azure-rbac-advisor.agent.md`. It answers RBAC questions grounded exclusively on the `resources/` library — it will not invent role names or recommend `Owner`/`Contributor` where a narrower role exists.

### How to Activate

#### GitHub Copilot CLI (Terminal)

1. **Install** the Copilot CLI (choose one):
   ```bash
   # macOS / Linux
   curl -fsSL https://gh.io/copilot-install | bash
   # or via Homebrew
   brew install copilot-cli

   # Windows (WinGet)
   winget install GitHub.Copilot

   # npm (all platforms)
   npm install -g @github/copilot
   ```

2. **Navigate** to this repository and **launch** the CLI:
   ```bash
   cd azure-rbac-agent-repo
   copilot
   ```

3. **Select the agent** using the `/agent` slash command:
   ```
   /agent
   ```
   Browse the list and select **Azure RBAC Advisor**.

4. **Ask your question.** The agent reads the `resources/` library, answers with exact role names, logs your prompt to `log/`, and saves the answer to `answer/` automatically.

5. **Switch modes** (optional) — press `Shift+Tab` to cycle to **Autopilot** mode for longer multi-step RBAC analysis without manual confirmation at each step.

> **Tip:** Run `copilot --experimental` to enable Autopilot mode, which lets the agent work through complex cross-resource questions uninterrupted.

> **Note:** Once you select a custom agent, it remains active for the duration of that chat session. To return to the default Copilot agent, start a new chat.

#### VS Code Copilot Chat

1. Open GitHub Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Click the agents dropdown (the `@` or Copilot icon at the bottom of the chat)
3. Select **Azure RBAC Advisor**
4. Ask your question

#### GitHub.com Copilot

1. Go to [github.com/copilot/agents](https://github.com/copilot/agents)
2. Select this repository and choose **Azure RBAC Advisor** from the agent list

### What the Agent Can Answer

- Least-privileged role for a specific operation on a specific resource
- Management plane vs. data plane role separation
- Sub-resource permission differences (e.g., Blob storage vs. File shares)
- Role assignments for Managed Identities and Service Principals
- Role comparisons across similar roles
- Scope recommendations (resource vs. resource group vs. subscription)
- Which resources are relevant to a given landing zone type

### Sample Prompts

**Single-resource lookup:**
```
What is the least-privileged role to read secrets from Azure Key Vault?
```

**Operation-specific:**
```
What role does a CI/CD pipeline need to push images to Azure Container Registry?
```

**Data plane separation:**
```
What is the difference between Storage Account Contributor and Storage Blob Data Contributor?
```

**Managed Identity scenario:**
```
A Managed Identity needs to read and write blobs in a Storage Account. What role should I assign and at what scope?
```

**Landing zone coverage:**
```
What are the least-privileged roles for deploying a new Azure Data Factory pipeline, including the integration runtime?
```

**Cross-resource summary (agent will offer to save to file):**
```
Give me a full RBAC summary for a Data Landing Zone covering Data Factory, Synapse, ADLS Gen2, and Databricks.
```

**Save output to file:**
```
What are the least-privileged roles for AKS operations? Save the answer to rbac-aks.md
```

---

### Full Workload Deployment Example

The following prompt asks the agent for a complete RBAC plan covering every layer of a typical three-tier web application workload deployed into a Workload Landing Zone. Run this from the CLI with the agent selected:

```
I'm deploying a three-tier web application into a Workload Landing Zone on Azure.
The workload uses the following resources:

  - Azure Virtual Network (spoke) with subnets and NSGs
  - Azure App Service (frontend + API tiers)
  - Azure SQL Database (backend data store)
  - Azure Storage Account (Blob for media uploads, Queue for async jobs)
  - Azure Key Vault (secrets and certificates for the app)
  - Azure Container Registry (container images for the API)
  - Azure Load Balancer and Application Gateway (traffic routing)

I need RBAC role assignments for FOUR principals:

  1. DevOps CI/CD pipeline (Service Principal) — needs to deploy and configure
     all resources above, push images to ACR, and manage App Service deployments.

  2. Application Managed Identity — needs runtime access to read secrets from
     Key Vault, read/write blobs in Storage, enqueue/dequeue Storage Queue
     messages, and query the SQL Database.

  3. Developer (individual user) — needs read-only access to view resource
     configurations and read application logs, but must NOT be able to modify
     production resources or access secret values.

  4. Database Administrator — needs full control of Azure SQL Database
     (server + databases), but no access to any other resource.

For each principal, list:
  - The least-privileged built-in role for each resource
  - Whether assignment should be at resource, resource group, or subscription scope
  - Any management plane vs. data plane role distinctions to be aware of

Save the full output to rbac-workload-deployment.md
```

This produces a structured RBAC assignment matrix saved to `rbac-workload-deployment.md`, with sources cited for each role from the `resources/workload-landing-zone/` reference files. The prompt is also automatically logged to `log/` and the answer saved to `answer/`.

### Guided Scoping Flow

If your question is broad or outside RBAC scope, the agent will guide you with three questions:
1. **What workload or landing zone type** are you working with?
2. **Which specific Azure resources** do you want to focus on?
3. **Do you want the output saved** to a file, and if so, what filename?

---

## Authoritative Sources

All RBAC role information in this library is drawn from official Microsoft documentation:

| Source | URL |
|---|---|
| Azure Built-in Roles | https://learn.microsoft.com/azure/role-based-access-control/built-in-roles |
| Azure RBAC Best Practices | https://learn.microsoft.com/azure/role-based-access-control/best-practices |
| Azure Resource Provider Operations | https://learn.microsoft.com/azure/role-based-access-control/resource-provider-operations |
| Microsoft Entra ID Roles | https://learn.microsoft.com/entra/identity/role-based-access-control/permissions-reference |
| Azure Landing Zones | https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/ |

---

## First-Time Setup

After cloning this repository, run the setup script to create the `log/` and `answer/` runtime directories used by the Azure RBAC Advisor agent:

```bash
# Option 1 — shell script
bash setup.sh

# Option 2 — make
make setup
```

These directories are excluded from git (via `.gitignore`) but must exist locally for the agent to write prompt logs and answer files.

---

## Contributing

When adding or modifying resource files, follow the authoring rules in [`.github/copilot-instructions.md`](.github/copilot-instructions.md). Every file must follow the mandatory 7-section structure with consistent RBAC table formatting and source citations.
