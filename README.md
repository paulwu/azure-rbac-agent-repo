# Azure RBAC Advisor Agent

A **GitHub Copilot custom agent** for generating least-privilege roles for specified workloads organized by Azure Landing Zone archetype. 
The project also serves as a repository for RBAC roles for common Azure resources. 

# Azure RBAC Knowledge Author

The repository also includes a second custom agent — the **Azure RBAC Knowledge Author** — for creating new resource RBAC reference files and validating existing ones. This agent generates the grounding knowledge that powers the Azure RBAC Advisor.

---

## Table of Contents

- [Purpose](#purpose)
- [Repository Structure](#repository-structure)
- [Using the Azure RBAC Advisor (Copilot Custom Agent)](#using-the-azure-rbac-advisor-copilot-custom-agent)
  - [How to Activate](#how-to-activate)
  - [Recommended Model](#recommended-model)
  - [What the Agent Can Answer](#what-the-agent-can-answer)
  - [Sample Prompts](#sample-prompts)
  - [Full Workload Deployment Example](#full-workload-deployment-example)
  - [Guided Scoping Flow](#guided-scoping-flow)
- [Using the Azure RBAC Knowledge Author](#using-the-azure-rbac-knowledge-author)
  - [Author Mode](#author-mode)
  - [Validate Mode](#validate-mode)
  - [Knowledge Author Sample Prompts](#knowledge-author-sample-prompts)
- [Test Use Cases](#test-use-cases)
- [Authoritative Sources](#authoritative-sources)
- [First-Time Setup](#first-time-setup)
- [Contributing](#contributing)

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
│   ├── platform-landing-zone/           # 23 resources (shared services, governance)
│   ├── workload-landing-zone/           # 23 resources (application workloads)
│   ├── data-landing-zone/               # 14 resources (data platforms and analytics)
│   └── ai-landing-zone/                 # 11 resources (AI/ML services)
├── .github/
│   ├── agents/
│   │   ├── azure-rbac-advisor.agent.md          # RBAC Advisor agent (query mode)
│   │   ├── azure-rbac-knowledge-author.agent.md # Knowledge Author agent (author + validate)
│   │   └── azure-rbac-test-runner.agent.md      # Test Runner agent (test use cases)
│   └── copilot-instructions.md          # Authoring rules for all contributions
├── test/
│   ├── use-case-01.md                   # CI/CD pipeline pushing images to ACR
│   └── use-case-02.md                   # Agentic AI Workload (16 resources)
└── README.md
```

### Resources by Landing Zone

| Landing Zone | Count | Resources Covered |
|---|---|---|
| **Platform** | 23 | Management Groups, Azure Policy, Log Analytics Workspace, Azure Monitor, Microsoft Defender for Cloud, Microsoft Sentinel, Azure Key Vault, Hub Virtual Network, Azure Firewall, Azure Front Door, Azure WAF Policy, Azure DDoS Protection Plan, Traffic Manager, VPN/ExpressRoute Gateway, Azure Bastion, Azure Arc, Private DNS Zones, Azure Automation Account, API Management, Managed Identity, Recovery Services Vault, Storage Sync Service, Azure SRE Agent *(preview)* |
| **Workload** | 23 | Spoke Virtual Network, Network Security Groups, Virtual Machines, Virtual Machine Scale Sets, Azure Managed Disks, Azure Virtual Desktop, App Service, App Service Plan, Azure SQL Database, Azure Database for PostgreSQL, Azure Storage Account *(with Blob/File/Queue/Table breakdowns)*, Azure Cache for Redis, Azure Key Vault, Azure Load Balancer, Application Gateway, Azure Container Registry, Azure Kubernetes Service, Azure Container Apps, Azure Container Apps Environment, Azure Functions, Azure Logic Apps, Azure Managed Grafana, Azure SSH Key |
| **Data** | 14 | Azure Data Factory, Azure Synapse Analytics, Azure Data Lake Storage Gen2, Azure Databricks, Azure Event Hubs, Azure Event Grid, Azure IoT Hub, Azure Cosmos DB, Azure Stream Analytics, Azure HDInsight, Azure Notification Hubs, Microsoft Purview, Microsoft Fabric, Azure Data Explorer |
| **AI** | 11 | Azure Machine Learning, Azure OpenAI, Azure AI Services, Azure AI Search, Azure Bot Service, Azure Applied AI Services, Azure AI Foundry, Azure AI Document Intelligence, Azure AI Content Understanding, Grounding with Bing Search, Foundry IQ |

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

### Recommended Model

For best results, select **Claude Opus 4.6 (1M context)(Internal only)** when running the Azure RBAC Advisor. Its large context window allows the agent to load the full `resources/` library in a single pass, ensuring role recommendations are grounded across all 71 resource files simultaneously — particularly important for complex cross-resource prompts covering many services at once (e.g., a full Agentic AI Workload with 16 resources).

**To select the model in the GitHub Copilot CLI:**

```bash
# Press 'm' or use the model picker at the prompt to switch models
# Select: Claude Opus 4.6 (1M context)(Internal only)
```

**To select the model in VS Code Copilot Chat:**

1. Click the model selector in the Copilot Chat panel (bottom of the input box)
2. Choose **Claude Opus 4.6 (1M context)(Internal only)**

> **Note:** The 1M context model is available to internal users only. If unavailable in your environment, **Claude Sonnet 4.6** is a capable fallback for most single-landing-zone queries.

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

## Using the Azure RBAC Knowledge Author

The **Azure RBAC Knowledge Author** is a second Copilot custom agent defined in `.github/agents/azure-rbac-knowledge-author.agent.md`. It creates new resource RBAC reference files and validates existing ones in `resources/`. Unlike the Advisor (which is read-only), the Knowledge Author writes to and modifies files in `resources/`.

### Author Mode

Create a new resource file by telling the agent which resource to add. The agent will:

1. Determine the correct landing zone and filename
2. Research the resource using official Microsoft documentation
3. Verify every role name against the [Azure built-in roles list](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
4. Generate the file following the mandatory 7-section structure
5. Update Related Resources links in existing files
6. Self-validate the new file before confirming

### Validate Mode

Check existing files for structural compliance and role name accuracy. The agent performs:

- **Structural checks** — all 7 mandatory sections present, correct heading levels (no H4+), no HTML, correct table column format
- **Role name verification** — every backtick-quoted role name checked against the Azure built-in roles list
- **Scope check** — flags `Subscription` scope where resource-level would suffice, flags `Owner`/`Contributor` where narrower roles exist

Validate a single file, an entire landing zone, or the full library.

### Knowledge Author Sample Prompts

**Add a new resource:**
```
Add Azure Service Bus to data-landing-zone
```

**Validate a single file:**
```
Validate azure-key-vault.md for accuracy
```

**Batch validate a landing zone:**
```
Validate all files in ai-landing-zone
```

**Validate everything:**
```
Validate all resources
```

**Fix validation issues:**
```
Fix all issues found in the validation report
```

---

## Test Use Cases

The `test/` folder contains reference use cases that validate agent output quality and demonstrate the range of questions the Azure RBAC Advisor can answer. Each file has two sections: the exact prompt used (`Section 1`) and the expected RBAC output (`Section 2`).

| File | Scenario | Resources |
|---|---|---|
| [`test/use-case-01.md`](test/use-case-01.md) | CI/CD pipeline pushing container images to Azure Container Registry | ACR |
| [`test/use-case-02.md`](test/use-case-02.md) | Agentic AI Workload on Microsoft Foundry with Private Endpoint isolation | 16 resources: AI Foundry, Foundry Project, Foundry IQ, Azure OpenAI, Azure AI Search, Azure AI Services, Document Intelligence, Azure Cosmos DB, Azure Storage, Azure Key Vault, Application Insights, Log Analytics Workspace, Virtual Network, Private Endpoints, Private DNS Zones, Azure DNS Private Resolver |

### How to Run a Use Case

#### Option 1 — Azure RBAC Test Runner agent (recommended)

The **Azure RBAC Test Runner** is a dedicated Copilot agent (`.github/agents/azure-rbac-test-runner.agent.md`) purpose-built for test execution. It independently generates answers using the same `resources/` grounding rules as the Advisor, scores them against expected output, and reports pass/fail — with full separation between the system under test and the test runner.

1. **Launch the CLI** and select the **Azure RBAC Test Runner** agent:
   ```bash
   cd azure-rbac-agent-repo
   copilot
   # then: /agent → Azure RBAC Test Runner
   ```

2. **Run a single test:**
   ```
   run-test @test/use-case-01.md
   ```

3. **Or batch-run all tests:**
   ```
   run-all-tests
   ```

4. The agent will:
   - Read the use case file and extract the prompt from `Section 1`
   - Execute the prompt as an RBAC query against the `resources/` library
   - Compare its output against the expected output in `Section 2`
   - Score the match (roles matched + key terms matched) and report a result:
     - **≥ 80%** → `✅ PASS`
     - **50–79%** → `⚠️ PARTIAL`
     - **< 50%** → `❌ FAIL`
   - For `run-all-tests`, produce a summary table with all results

> **Note:** Test runs do not log to `log/` or save to `answer/` — they are kept separate from normal Advisor interactions.

#### Option 2 — Manual copy-paste via the Advisor

You can also run use cases manually by copying the prompt into the Azure RBAC Advisor agent:

1. **Launch the CLI** and select the **Azure RBAC Advisor** agent.

2. **Copy the prompt** from `Section 1` of the use case file.

3. **Paste it** into the agent chat and press Enter.

4. **Compare** the agent's response against the expected output in `Section 2` manually.

> **Tip:** Switch to **Autopilot mode** (`Shift+Tab`) before running multi-resource use cases like `use-case-02.md` — the agent will process all 16 resources in a single uninterrupted pass and save the full output to `answer/`.

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

After cloning this repository, run the setup command to create the `log/` and `answer/` runtime directories used by the Azure RBAC Advisor agent:

```bash
make setup
```

`log/` is gitignored. `answer/` is tracked in git (answer files are committed so output quality can be compared over time). Both directories must exist locally for the Advisor agent to write prompt logs and answer files.

---

## Contributing

When adding or modifying resource files, follow the authoring rules in [`.github/copilot-instructions.md`](.github/copilot-instructions.md). Every file must follow the mandatory 7-section structure with consistent RBAC table formatting and source citations.

The recommended workflow for adding new resources is to use the **Azure RBAC Knowledge Author** agent, which automates file generation, role verification, cross-reference updates, and self-validation. To add a resource manually, ensure you:

1. Place the file in the correct landing zone directory
2. Follow the 7-section structure (Resource Metadata, Overview, Least-Privilege RBAC Reference with 🟢🟡🔴⚙️, Runtime Dependencies, Notes / Considerations, Related Resources)
3. Verify every role name against the [Azure built-in roles list](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
4. Update Related Resources links in connected files
5. Run the Knowledge Author in validate mode to check your work
