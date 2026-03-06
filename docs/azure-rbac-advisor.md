# Azure RBAC Advisor — User Guide

The **Azure RBAC Advisor** is a GitHub Copilot custom agent defined in `.github/agents/azure-rbac-advisor.agent.md`. It answers RBAC questions grounded exclusively on the `resources/` library — it will not invent role names or recommend `Owner`/`Contributor` where a narrower role exists.

---

## How to Activate

### GitHub Copilot CLI (Terminal)

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

### VS Code Copilot Chat

1. Open GitHub Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Click the agents dropdown (the `@` or Copilot icon at the bottom of the chat)
3. Select **Azure RBAC Advisor**
4. Ask your question

### GitHub.com Copilot

1. Go to [github.com/copilot/agents](https://github.com/copilot/agents)
2. Select this repository and choose **Azure RBAC Advisor** from the agent list

---

## Recommended Model

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

---

## What the Agent Can Answer

- Least-privileged role for a specific operation on a specific resource
- Management plane vs. data plane role separation
- Sub-resource permission differences (e.g., Blob storage vs. File shares)
- Role assignments for Managed Identities and Service Principals
- Role comparisons across similar roles
- Scope recommendations (resource vs. resource group vs. subscription)
- Which resources are relevant to a given landing zone type

---

## Sample Prompts

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

## Full Workload Deployment Example

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

---

## Guided Scoping Flow

If your question is broad or outside RBAC scope, the agent will guide you with three questions:
1. **What workload or landing zone type** are you working with?
2. **Which specific Azure resources** do you want to focus on?
3. **Do you want the output saved** to a file, and if so, what filename?
