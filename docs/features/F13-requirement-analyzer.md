# F13: Requirement Analyzer (`unit-test-requirement-analyzer`)

## 1. Purpose

Turn business requirements (Jira issues, or exported requirement files) into testable rules (`REQ-xxx`), concrete scenarios with expected results (`SCN-xxx`), and open questions (`UNC-xxx`), **without ever reading the code under test**.

This solves the test oracle problem: when tests are derived only from the code, a wrong implementation produces tests that pass and lock in the wrong behavior. Expected results must come from the requirement; the code is only the thing under test. The generator ([F02](F02-generator-generate.md)) turns these scenarios into tests, and the reviewer ([F04](F04-reviewer.md)) compares requirement, tests, and code.

Requirements are **optional**. Without them, the implementation (with its docs and types) is treated as the requirement, and every report says so.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                                    | Headless / CLI Invocation                                                   |
| :----------------- | :-------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **Claude Code**    | `/unit-test-requirement-analyzer --jira SHOP-142`         | `claude -p "/unit-test-requirement-analyzer --jira SHOP-142"`               |
| **Codex CLI**      | `$unit-test-requirement-analyzer --jira SHOP-142`         | `codex exec "$unit-test-requirement-analyzer --jira SHOP-142"`              |
| **Gemini CLI**     | `/unit-test-requirement-analyzer --jira SHOP-142`         | `gemini -p "/unit-test-requirement-analyzer --jira SHOP-142"`               |
| **GitHub Copilot** | `/unit-test-requirement-analyzer --jira SHOP-142`         | N/A (VS Code Chat Agent mode)                                               |

| Argument                         | Meaning                                                                                                   |
| :------------------------------- | :-------------------------------------------------------------------------------------------------------- |
| `--jira <KEY>[,<KEY>...]`        | Primary source. Jira issue keys read through the Jira MCP server configured in the runtime.              |
| `--req-file <path>[,<path>...]`  | Offline source: Markdown/text exports of requirements. Used by fixtures, `evals/`, and teams without Jira. |

Exactly one of `--jira` or `--req-file` is required.

**Context isolation:** Every runtime wrapper runs this agent in its own fresh subagent ([F08](F08-multi-runtime-packaging.md)), the same way as the reviewer, so it never shares context with a step that has read the source code.

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has a supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
2. `--jira`: a Jira MCP server is available in the runtime (for example Atlassian's remote MCP server with a `getJiraIssue` tool). If not, stop and point to the README section "Requirements from Jira".
3. `--jira`: every key is readable. If any key is missing or forbidden, stop and list the keys that failed.
4. `--req-file`: every file exists and is outside `sourceRoots` and `testRoots`.

### Isolation & Read-Only Rules

| Rule                     | Requirement                                                                                                     |
| :----------------------- | :-------------------------------------------------------------------------------------------------------------- |
| Never read code          | Never open, read, search, or list files under `sourceRoots` or `testRoots`. Read `project.json` only for preconditions and to know which paths to avoid. |
| Jira is read-only        | Only read issues. Never create, edit, comment on, transition, link, or log work on any Jira issue.             |
| Fields read              | Summary, description, the acceptance-criteria field (any field whose name contains "Acceptance Criteria"; if absent, an "Acceptance Criteria" heading in the description), and comments. Do not follow linked issues, sub-tasks, attachments, or Confluence links. |
| No invented values       | Every expected result must be stated in, or follow by arithmetic from, the requirement text. Otherwise raise an `UNC` item. |
| Keep the source language | Quote requirement text verbatim; do not translate it.                                                           |

### Outputs

- **Path:** `.unit-test-agent/runs/<run-id>/requirements.json` (this agent creates the `<run-id>`; slug = first Jira key or file name, e.g., `20261105-101500-shop-142`).
- **Schema:** `.unit-test-agent/schemas/requirements.schema.json` ([F07](F07-json-contracts.md)).

### Complete JSON Example (`.unit-test-agent/runs/20261105-101500-shop-142/requirements.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "unit-test-requirement-analyzer",
  "runId": "20261105-101500-shop-142",
  "createdAt": "2026-11-05T10:15:00Z",
  "sources": [
    {
      "kind": "jira",
      "ref": "SHOP-142",
      "title": "Discount for large orders",
      "fieldsRead": ["summary", "description", "Acceptance Criteria", "comments"],
      "fetchedAt": "2026-11-05T10:15:00Z",
      "snapshot": "Summary: Discount for large orders\nAcceptance Criteria:\n1. Orders of $100 or more get a 10% discount.\n2. Orders under $100 get no discount.\n3. Negative order amounts are rejected with an error."
    }
  ],
  "requirements": [
    {
      "id": "REQ-001",
      "sourceRef": "SHOP-142",
      "sourceQuote": "Orders of $100 or more get a 10% discount. Orders under $100 get no discount.",
      "statement": "A 10% discount applies when the order amount is at least $100; below $100 there is no discount.",
      "scenarios": [
        { "id": "SCN-001", "category": "boundary", "given": "order amount is $99.99", "expected": "discount is $0" },
        { "id": "SCN-002", "category": "boundary", "given": "order amount is exactly $100", "expected": "discount is $10" },
        { "id": "SCN-003", "category": "happy_path", "given": "order amount is $200", "expected": "discount is $20" },
        { "id": "SCN-004", "category": "edge", "given": "order amount is $0", "expected": "discount is $0" }
      ]
    },
    {
      "id": "REQ-002",
      "sourceRef": "SHOP-142",
      "sourceQuote": "Negative order amounts are rejected with an error.",
      "statement": "A negative order amount is rejected with an error.",
      "scenarios": [
        { "id": "SCN-005", "category": "error", "given": "order amount is -$1", "expected": "an error is raised" }
      ]
    }
  ],
  "unclear": [
    {
      "id": "UNC-001",
      "sourceRef": "SHOP-142",
      "requirementIds": ["REQ-001"],
      "question": "Is the discount rounded to cents, and if so, half-up or half-even?",
      "reason": "The criteria give a percentage but no rounding rule; $100.05 gives a discount of $10.005."
    }
  ]
}
```

---

## 4. Step-by-Step Behavior

1. **Check Preconditions** (Section 3). Stop on the first failure and say what the user must do.
2. **Read & Snapshot Sources:** For each key or file, read the allowed fields and store the exact text read in `sources[].snapshot`, so later agents and reports never need Jira again.
3. **Extract Requirements:** Create one `REQ` per testable business rule (a condition and its outcome). Record the verbatim `sourceQuote` it came from.
4. **Design Scenarios per Requirement:**

   | Category     | Write a scenario for                                                                                  |
   | :----------- | :---------------------------------------------------------------------------------------------------- |
   | `happy_path` | A typical input that satisfies the rule.                                                              |
   | `boundary`   | Each threshold the text states: exactly at it, and just below/above it (smallest step the domain allows). |
   | `edge`       | Zero, empty, or minimum inputs the text covers.                                                       |
   | `error`      | Each rejection or error the text states.                                                              |

   - Write `given` and `expected` in business terms from the requirement (amounts, units, states). Never use function names, parameters, or code types: this agent has not seen them.
5. **Raise Unclear Items:** Create an `UNC` item (and no scenario for the ambiguous part) when the text leaves the expected result open: inclusive vs exclusive thresholds, missing rounding or unit rules, undefined error behavior, or conflicts between description, acceptance criteria, and comments.
6. **Write Output:** Write `requirements.json` and report the `<run-id>`, the number of requirements, scenarios, and unclear items.

---

## 5. Edge Cases

- **No acceptance-criteria field:** Use the description. If no testable rule exists anywhere, write zero requirements and one `UNC` item explaining why.
- **Comment contradicts the acceptance criteria:** Do not pick a side; raise an `UNC` item quoting both.
- **Several keys describe the same rule:** Keep one `REQ` and cite the clearest source in `sourceRef`.
- **Jira MCP not configured or not authorized:** Stop before creating the run directory; point to the README setup section.

---

## 6. Acceptance Criteria Checklist

- [ ] Runs in a fresh, isolated subagent and never reads files under `sourceRoots` or `testRoots`.
- [ ] Performs zero write operations on Jira.
- [ ] Stops with a clear message when the Jira MCP server is unavailable or a key cannot be read.
- [ ] Every `REQ` has a verbatim `sourceQuote`; every `sources[]` entry has a `snapshot`.
- [ ] Every threshold stated in the requirement has boundary scenarios at and next to it.
- [ ] Ambiguities produce `UNC` items instead of guessed expected results.
- [ ] Writes a valid `requirements.json` conforming to `schemas/requirements.schema.json`.
