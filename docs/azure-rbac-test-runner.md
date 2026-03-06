# Azure RBAC Test Runner — User Guide

The **Azure RBAC Test Runner** is a dedicated Copilot custom agent defined in `.github/agents/azure-rbac-test-runner.agent.md`. It validates the quality of RBAC role recommendations by executing test use case prompts against the `resources/` reference library and scoring the output against expected results.

The Test Runner operates independently from the Advisor — it uses the same `resources/` grounding rules but runs in its own context, providing separation between the system under test and the test runner.

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
   Browse the list and select **Azure RBAC Test Runner**.

3. Run a test using the commands below.

### VS Code Copilot Chat

1. Open GitHub Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Click the agents dropdown and select **Azure RBAC Test Runner**
3. Send a `run-test` or `run-all-tests` command

---

## Commands

### `run-test` — Single Test

Run a single use case file:

```
run-test @test/use-case-01.md
```

The agent will:
1. Read the use case file and extract the prompt from `Section 1`
2. Execute the prompt as an RBAC query against the `resources/` library
3. Compare its output against the expected output in `Section 2`
4. Score the match (roles matched + key terms matched) and report a result:
   - **≥ 80%** → `✅ PASS`
   - **50–79%** → `⚠️ PARTIAL`
   - **< 50%** → `❌ FAIL`

### `run-all-tests` — Batch Test

Run all use case files in the `test/` folder:

```
run-all-tests
```

The agent discovers all `test/use-case-*.md` files, runs each one, and produces a summary table:

```markdown
## 🧪 Test Suite Summary

| Test File | Score | Result |
|---|---|---|
| `test/use-case-01.md` | XX% | ✅ PASS / ⚠️ PARTIAL / ❌ FAIL |
| `test/use-case-02.md` | XX% | ✅ PASS / ⚠️ PARTIAL / ❌ FAIL |

**Overall: X/Y passed**
```

---

## Test Use Case Format

Each test file in `test/` follows this structure:

```markdown
# Use Case XX: [Scenario Title]

## Section 1 — Prompt

[The exact prompt to execute as an RBAC query]

## Section 2 — Expected Output

[The reference answer containing expected role names, scopes, and key terms]
```

---

## Scoring Method

The Test Runner compares Actual Output against Expected Output using:

1. **Role matching** — extracts all backtick-quoted Azure role names from both outputs and counts matches
2. **Key term matching** — extracts resource names, scope values, and directive terms from Expected Output and checks for their presence in Actual Output
3. **Score calculation** — `(roles_matched + terms_matched) / (total_roles + total_terms) × 100`

---

## Available Test Cases

| File | Scenario | Resources |
|---|---|---|
| [`test/use-case-01.md`](../test/use-case-01.md) | CI/CD pipeline pushing container images to Azure Container Registry | ACR |
| [`test/use-case-02.md`](../test/use-case-02.md) | Agentic AI Workload on Microsoft Foundry with Private Endpoint isolation | 16 resources |

---

## Constraints

- Test runs do **not** log to `log/` or save to `answer/` — they are kept separate from normal Advisor interactions
- The agent does **not** answer general RBAC questions — use the [Azure RBAC Advisor](./azure-rbac-advisor.md) for that
- The agent does **not** modify any files in `test/` or `resources/`
