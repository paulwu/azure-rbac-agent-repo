# Azure RBAC Knowledge Author — User Guide

The **Azure RBAC Knowledge Author** is a Copilot custom agent defined in `.github/agents/azure-rbac-knowledge-author.agent.md`. It creates new resource RBAC reference files and validates existing ones in `resources/`. Unlike the Advisor (which is read-only), the Knowledge Author writes to and modifies files in `resources/`.

---

## Author Mode

Create a new resource file by telling the agent which resource to add. The agent will:

1. Determine the correct landing zone and filename
2. Research the resource using official Microsoft documentation
3. Verify every role name against the [Azure built-in roles list](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
4. Generate the file following the mandatory 7-section structure
5. Update Related Resources links in existing files
6. Self-validate the new file before confirming

---

## Validate Mode

Check existing files for structural compliance and role name accuracy. The agent performs:

- **Structural checks** — all 7 mandatory sections present, correct heading levels (no H4+), no HTML, correct table column format
- **Role name verification** — every backtick-quoted role name checked against the Azure built-in roles list
- **Scope check** — flags `Subscription` scope where resource-level would suffice, flags `Owner`/`Contributor` where narrower roles exist

Validate a single file, an entire landing zone, or the full library.

---

## How to Activate

### GitHub Copilot CLI (Terminal)

1. Navigate to this repository and launch the CLI:
   ```bash
   cd azure-rbac-agent-repo
   copilot
   ```

2. Select the agent:
   ```
   /agent
   ```
   Browse the list and select **Azure RBAC Knowledge Author**.

3. Tell the agent what to do (author a new resource or validate existing files).

### VS Code Copilot Chat

1. Open GitHub Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Click the agents dropdown and select **Azure RBAC Knowledge Author**
3. Send your prompt

---

## Sample Prompts

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
