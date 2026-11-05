# F13: Requirement Analyzer (`unit-test-requirement-analyzer`)

## 1. Purpose

Turn business requirements (Jira issues, or exported requirement files) into testable rules (`REQ-xxx`), concrete scenarios with expected results (`SCN-xxx`), and open questions (`UNC-xxx`), **without ever reading the bodies of the code under test**.

This solves the test oracle problem: when tests are derived only from the code, a wrong implementation produces tests that pass and lock in the wrong behavior. Expected results must come from the requirement; the code is only the thing under test. The generator ([F02](F02-generator-generate.md)) turns these scenarios into tests, and the reviewer ([F04](F04-reviewer.md)) compares requirement, tests, and code.

To write concrete calls (`calculateDiscount(100) → 10`), the agent may see the **public signatures** of the targets: names, parameter types, return types. It gets them only from a read-only signature extractor (Section 4), never by opening source files, so bodies, default values, comments, and constant values never reach its context.

Requirements are **optional**. Without them, the implementation (with its docs and types) is treated as the requirement, and every report says so.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation                                                   | Headless / CLI Invocation                                                            |
| :----------------- | :----------------------------------------------------------------------- | :----------------------------------------------------------------------------------- |
| **Claude Code**    | `/unit-test-requirement-analyzer src/discount.ts --jira SHOP-142`        | `claude -p "/unit-test-requirement-analyzer src/discount.ts --jira SHOP-142"`        |
| **Codex CLI**      | `$unit-test-requirement-analyzer src/discount.ts --jira SHOP-142`        | `codex exec "$unit-test-requirement-analyzer src/discount.ts --jira SHOP-142"`       |
| **Gemini CLI**     | `/unit-test-requirement-analyzer src/discount.ts --jira SHOP-142`        | `gemini -p "/unit-test-requirement-analyzer src/discount.ts --jira SHOP-142"`        |
| **GitHub Copilot** | `/unit-test-requirement-analyzer src/discount.ts --jira SHOP-142`        | N/A (VS Code Chat Agent mode)                                                        |

| Argument                         | Meaning                                                                                                   |
| :------------------------------- | :-------------------------------------------------------------------------------------------------------- |
| `[targets]`                      | Optional. Files, folders, or globs under `sourceRoots`. Passed only to the signature extractor. Without targets, scenarios stay in business terms (strict mode). |
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
5. `[targets]`: resolve to files under `sourceRoots`. A missing or failing extractor is **not** a precondition failure; it switches to strict mode (Section 4).

### Isolation & Read-Only Rules

| Rule                       | Requirement                                                                                                     |
| :------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| Code only via extractor    | Never open, read, search, or list files under `sourceRoots` or `testRoots`. The only view of the code is the stdout of the signature extractor. Read `project.json` only for preconditions, `language`, and the paths to avoid. |
| Names are not requirements | Never take an expected value, threshold, or rule from a symbol, parameter, or type name (e.g., `applyDiscountOver200`). Values come from the requirement text only. |
| Jira is read-only          | Only read issues. Never create, edit, comment on, transition, link, or log work on any Jira issue.             |
| Fields read                | Summary, description, the acceptance-criteria field (any field whose name contains "Acceptance Criteria"; if absent, an "Acceptance Criteria" heading in the description), and comments. Do not follow linked issues, sub-tasks, attachments, or Confluence links. |
| No invented values         | Every expected result must be stated in, or follow by arithmetic from, the requirement text. Otherwise raise an `UNC` item. |
| Keep the source language   | Quote requirement text verbatim; do not translate it.                                                           |

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
  "codeContract": {
    "status": "extracted",
    "targets": ["src/discount.ts"],
    "commandExecuted": "node .unit-test-agent/tools/extract-signatures.mjs src/discount.ts",
    "reason": null,
    "symbols": [
      {
        "file": "src/discount.ts",
        "name": "calculateDiscount",
        "kind": "function",
        "signature": "calculateDiscount(amount: number): number"
      }
    ]
  },
  "requirements": [
    {
      "id": "REQ-001",
      "sourceRef": "SHOP-142",
      "sourceQuote": "Orders of $100 or more get a 10% discount. Orders under $100 get no discount.",
      "statement": "A 10% discount applies when the order amount is at least $100; below $100 there is no discount.",
      "scenarios": [
        { "id": "SCN-001", "category": "boundary", "given": "order amount is $99.99", "expected": "discount is $0", "call": "calculateDiscount(99.99)", "expectedValue": "0" },
        { "id": "SCN-002", "category": "boundary", "given": "order amount is exactly $100", "expected": "discount is $10", "call": "calculateDiscount(100)", "expectedValue": "10" },
        { "id": "SCN-003", "category": "happy_path", "given": "order amount is $200", "expected": "discount is $20", "call": "calculateDiscount(200)", "expectedValue": "20" },
        { "id": "SCN-004", "category": "edge", "given": "order amount is $0", "expected": "discount is $0", "call": "calculateDiscount(0)", "expectedValue": "0" }
      ]
    },
    {
      "id": "REQ-002",
      "sourceRef": "SHOP-142",
      "sourceQuote": "Negative order amounts are rejected with an error.",
      "statement": "A negative order amount is rejected with an error.",
      "scenarios": [
        { "id": "SCN-005", "category": "error", "given": "order amount is -$1", "expected": "an error is raised", "call": "calculateDiscount(-1)", "expectedValue": "throws" }
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

In strict mode (`codeContract.status` is `"not_requested"` or `"unavailable"`), `symbols` is `[]` and every scenario has `"call": null` and `"expectedValue": null`; the generator maps them ([F02](F02-generator-generate.md)).

---

## 4. Signature Extractors (`tools/`)

The only scripts this project installs into user repositories (see `AGENTS.md`). They are read-only, deterministic, and contain no agent logic.

| Ecosystem       | Extractor (installed under `.unit-test-agent/tools/`) | Command                                                       | Runtime / parser                                   |
| :-------------- | :---------------------------------------------------- | :------------------------------------------------------------ | :------------------------------------------------- |
| JS / TS         | `extract-signatures.mjs`                              | `node .unit-test-agent/tools/extract-signatures.mjs <files…>` | Node + the repo's own `typescript` package         |
| Python          | `extract_signatures.py`                               | `python .unit-test-agent/tools/extract_signatures.py <files…>` | Python 3.9+ standard library `ast`                |
| Java/Kotlin, C#/.NET, Go, Rust | None yet                               | —                                                             | Strict mode until an extractor is added            |

| Extractor rule   | Requirement                                                                                                         |
| :--------------- | :------------------------------------------------------------------------------------------------------------------ |
| Keeps            | Exported functions, classes, public methods, and type declarations: names, parameter names and types, optional markers, return types, enum member names, literal-union types. |
| Drops            | Function and method bodies, parameter default values (`threshold = 200` → `threshold?: number`), comments, JSDoc, docstrings, constant initializers (`const MIN = 200` → `const MIN: number`), enum member values, decorators' arguments. |
| Dependencies     | Uses only the target repo's runtime and already-installed packages. No network. No new dependencies.               |
| Side effects     | Reads only the files passed as arguments. Writes nothing; prints JSON to stdout.                                    |
| Output (stdout)  | `{ "symbols": [ { "file", "name", "kind", "signature" } ] }`, exit code `0`.                                        |
| Cannot run       | Exit code `2` with one line on stderr (e.g., `typescript package not found`). The agent switches to strict mode.   |

---

## 5. Step-by-Step Behavior

1. **Check Preconditions** (Section 3). Stop on the first failure and say what the user must do.
2. **Read & Snapshot Sources:** For each key or file, read the allowed fields and store the exact text read in `sources[].snapshot`, so later agents and reports never need Jira again.
3. **Extract the Code Contract:**
   - No `[targets]`: `codeContract.status = "not_requested"`.
   - Otherwise pick the extractor for `project.json` `language`. Run it on the resolved target files and store its symbols with `status: "extracted"` and `commandExecuted`.
   - No extractor for the ecosystem, or exit code `2`: `status: "unavailable"` with `reason`; continue in strict mode.
4. **Extract Requirements:** Create one `REQ` per testable business rule (a condition and its outcome). Record the verbatim `sourceQuote` it came from.
5. **Design Scenarios per Requirement:**

   | Category     | Write a scenario for                                                                                  |
   | :----------- | :---------------------------------------------------------------------------------------------------- |
   | `happy_path` | A typical input that satisfies the rule.                                                              |
   | `boundary`   | Each threshold the text states: exactly at it, and just below/above it (smallest step the domain allows). |
   | `edge`       | Zero, empty, or minimum inputs the text covers.                                                       |
   | `error`      | Each rejection or error the text states.                                                              |

   - Always write `given` and `expected` in business terms from the requirement.
   - When a symbol in `codeContract` clearly implements the rule, also write `call` (a call using only that symbol's signature) and `expectedValue` (the requirement's result in the return type's form, or `throws`). Otherwise set both to `null`.
6. **Raise Unclear Items:** Create an `UNC` item (and no scenario for the ambiguous part) when the text leaves the expected result open: inclusive vs exclusive thresholds, missing rounding or unit rules, undefined error behavior, conflicts between description, acceptance criteria, and comments, or a requirement whose inputs no extracted signature can express.
7. **Write Output:** Write `requirements.json` and report the `<run-id>`, the number of requirements, scenarios, and unclear items, and the `codeContract.status`.

---

## 6. Edge Cases

- **No acceptance-criteria field:** Use the description. If no testable rule exists anywhere, write zero requirements and one `UNC` item explaining why.
- **Comment contradicts the acceptance criteria:** Do not pick a side; raise an `UNC` item quoting both.
- **Several keys describe the same rule:** Keep one `REQ` and cite the clearest source in `sourceRef`.
- **Jira MCP not configured or not authorized:** Stop before creating the run directory; point to the README setup section.
- **Plain JavaScript repo without `typescript` installed:** Extractor exits `2`; strict mode.
- **Unit mismatch (requirement in dollars, signature `amountCents: number`):** Convert in `call` and `expectedValue` (`calculateDiscount(10000)` → `1000`) and keep `given`/`expected` in the requirement's units.
- **Signatures change later:** Re-run this agent; `requirements.json` is tied to the signatures it recorded.

---

## 7. Acceptance Criteria Checklist

- [ ] Runs in a fresh, isolated subagent and never opens files under `sourceRoots` or `testRoots`; its only view of code is extractor stdout.
- [ ] Performs zero write operations on Jira.
- [ ] Stops with a clear message when the Jira MCP server is unavailable or a key cannot be read.
- [ ] Records `codeContract.status` and falls back to strict mode (`call`/`expectedValue` `null`) when no extractor can run.
- [ ] Every `REQ` has a verbatim `sourceQuote`; every `sources[]` entry has a `snapshot`.
- [ ] Every threshold stated in the requirement has boundary scenarios at and next to it.
- [ ] Ambiguities produce `UNC` items instead of guessed expected results.
- [ ] Both extractors drop bodies, default values, comments, and constant values (verified by `CHK-16`).
- [ ] Writes a valid `requirements.json` conforming to `schemas/requirements.schema.json`.
