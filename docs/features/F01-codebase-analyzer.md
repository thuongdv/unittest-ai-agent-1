# F01: Codebase Analyzer (`codebase-analyzer`)

## 1. Purpose

Inspect the target repository automatically, verify the existing test baseline by running the detected test suite, and write `.unit-test-agent/project.json`. This file serves as the shared configuration contract for all downstream agents (`unit-test-generator`, `unit-test-reviewer`, and `unit-test-failure-analyzer`).

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation | Headless / CLI Invocation         | Arguments                                                            |
| :----------------- | :--------------------- | :-------------------------------- | :------------------------------------------------------------------- |
| **Claude Code**    | `/codebase-analyzer`   | `claude -p "/codebase-analyzer"`  | Optional: `[subproject-path]` (for scoping analysis or re-verifying) |
| **Codex CLI**      | `$codebase-analyzer`   | `codex exec "$codebase-analyzer"` | Optional: `[subproject-path]`                                        |
| **Gemini CLI**     | `/codebase-analyzer`   | `gemini -p "/codebase-analyzer"`  | Optional: `[subproject-path]`                                        |
| **GitHub Copilot** | `/codebase-analyzer`   | N/A (VS Code Chat Agent mode)     | Optional: `[subproject-path]`                                        |

---

## 3. Inputs and Outputs

### Inputs

- **Target repository files:** Manifests (`package.json`, `pyproject.toml`, `pom.xml`, `build.gradle`, `*.csproj`, `go.mod`, `Cargo.toml`), lockfiles, linter/typechecker configs, CI workflows (`.github/workflows/*`, `.gitlab-ci.yml`), and existing test files.
- **Existing `.unit-test-agent/project.json` (if re-running):** Used to preserve any user-edited values.

### Output

- **Path:** `.unit-test-agent/project.json`
- **Schema:** `.unit-test-agent/schemas/project.schema.json` (defined in [F07](F07-json-contracts.md))

### Complete JSON Example (`.unit-test-agent/project.json`)

```json
{
  "schemaVersion": "1.0.0",
  "agent": "codebase-analyzer",
  "createdAt": "2026-10-20T09:00:00Z",
  "updatedAt": "2026-10-20T09:00:00Z",
  "language": {
    "value": "typescript",
    "confidence": "high",
    "evidence": "tsconfig.json, package.json (devDependencies.typescript)"
  },
  "runtimeVersion": {
    "value": "node >=20.0.0",
    "confidence": "high",
    "evidence": "package.json#engines.node"
  },
  "packageManager": {
    "value": "npm",
    "confidence": "high",
    "evidence": "package-lock.json"
  },
  "commands": {
    "install": {
      "value": "npm ci",
      "confidence": "high",
      "evidence": "package-lock.json"
    },
    "typecheck": {
      "value": "npm run typecheck",
      "confidence": "high",
      "evidence": "package.json#scripts.typecheck"
    },
    "lint": {
      "value": "npm run lint",
      "confidence": "high",
      "evidence": "package.json#scripts.lint"
    },
    "test": {
      "value": "npx vitest run",
      "confidence": "high",
      "evidence": "package.json#scripts.test, vitest.config.ts"
    },
    "coverage": {
      "value": "npx vitest run --coverage",
      "confidence": "high",
      "evidence": "package.json#devDependencies.@vitest/coverage-v8"
    },
    "mutation": {
      "value": null,
      "confidence": "low",
      "evidence": "No Stryker or mutation tool found in package.json"
    }
  },
  "testFramework": {
    "value": "vitest",
    "confidence": "high",
    "evidence": "vitest.config.ts"
  },
  "mockingLibrary": {
    "value": "vitest (vi)",
    "confidence": "high",
    "evidence": "tests/cart.test.ts"
  },
  "coverageFormat": {
    "value": "json-summary",
    "confidence": "medium",
    "evidence": "vitest.config.ts (default v8 coverage reporters)"
  },
  "coverageReportPath": {
    "value": "coverage/coverage-final.json",
    "confidence": "medium",
    "evidence": "vitest.config.ts"
  },
  "sourceRoots": {
    "value": ["src"],
    "confidence": "high",
    "evidence": "tsconfig.json#include"
  },
  "testRoots": {
    "value": ["tests"],
    "confidence": "high",
    "evidence": "vitest.config.ts#test.include"
  },
  "testFileNamingPattern": {
    "value": "tests/**/*.test.ts",
    "confidence": "high",
    "evidence": "tests/cart.test.ts"
  },
  "exemplarTestFiles": {
    "value": ["tests/cart.test.ts"],
    "confidence": "high",
    "evidence": "Existing passing unit test file following repository conventions"
  },
  "mutationTool": {
    "value": null,
    "confidence": "low",
    "evidence": "No @stryker-mutator/core dependency found"
  },
  "monorepoSubprojects": {
    "value": [],
    "confidence": "high",
    "evidence": "Single root package.json without workspaces"
  },
  "ignoreGlobs": {
    "value": ["node_modules/**", "dist/**", "coverage/**", "**/*.d.ts"],
    "confidence": "high",
    "evidence": ".gitignore, tsconfig.json"
  },
  "policy": {
    "coverageThresholds": {
      "lines": 80,
      "branches": 70
    },
    "maxRepairAttempts": 3
  },
  "baseline": {
    "executedCommand": "npx vitest run",
    "exitCode": 0,
    "passed": 4,
    "failed": 0,
    "skipped": 0,
    "alreadyFailing": false,
    "summary": "All 4 baseline tests in tests/cart.test.ts passed."
  },
  "missing": [
    {
      "field": "mutationTool",
      "question": "No mutation testing tool (e.g., @stryker-mutator/core) is installed. Would you like to configure one, or skip mutation testing during reviews?",
      "blocking": false
    }
  ]
}
```

---

## 4. Step-by-Step Behavior

1. **Check Existing Configuration:**
   - Check if `.unit-test-agent/project.json` already exists.
   - If it exists, load it into memory. Any field where `.value` is already non-null (or `policy` fields modified by the user) **must not be overwritten** on re-run.
2. **Scan Repository Manifests & Ecosystem Reference Table:**
   - Inspect root and subdirectory manifests across supported ecosystems:
     - **JS/TS:** `package.json`, `tsconfig.json`, `vitest.config.*`, `jest.config.*`, `stryker.config.*`
     - **Python:** `pyproject.toml`, `pytest.ini`, `setup.cfg`, `tox.ini`, `.coveragerc` (`pytest`, `unittest`, `coverage.py`, `mutmut`)
     - **Java/Kotlin:** `pom.xml`, `build.gradle(.kts)` (`JUnit 5`, `Mockito`/`MockK`, `JaCoCo`, `PIT`)
     - **C#/.NET:** `*.sln`, `*.csproj` (`xUnit`/`NUnit`/`MSTest`, `Moq`/`NSubstitute`, `Coverlet`, `Stryker.NET`)
     - **Go:** `go.mod` (`go test`, `testify`, `go test -coverprofile`, `go-mutesting`/`gremlins`)
     - **Rust:** `Cargo.toml` (`cargo test`, `mockall`, `cargo-llvm-cov`/`tarpaulin`, `cargo-mutants`)
3. **Select Style Exemplars:**
   - Locate 2–3 representative existing test files (or fewer if the repo has <2 test files) that demonstrate assertion style, mocking conventions, and describe/test block structure.
4. **Verify by Running Baseline Tests:**
   - Run the detected `commands.test.value` command.
   - Record exit code, pass count, fail count, skipped count, and `alreadyFailing` (`true` if exit code != 0 or failed > 0).
   - If baseline tests fail, suggest running `/unit-test-failure-analyzer` in the console summary, and record `alreadyFailing: true`.
5. **Populate `missing` Array:**
   - For any field whose `value` is `null` or `confidence` is `low`, add an entry to `missing` with `field`, a clear human-readable `question`, and `blocking: true | false`.
   - **Blocking (`blocking: true`):** Missing `language`, `testFramework`, `commands.test`, `sourceRoots`, or `testRoots`.
   - **Non-blocking (`blocking: false`):** Missing `mutationTool`, `commands.mutation`, `commands.lint`, `commands.typecheck`, or `exemplarTestFiles` (when the repo has zero existing tests).
6. **Write Output & Summarize:**
   - Write `.unit-test-agent/project.json` formatted with 2-space indentation.
   - Print a concise summary of detected settings, baseline test status, and any `missing` questions.

---

## 5. Edge Cases

- **Re-run after user edits:** User manually changes `policy.coverageThresholds.lines` to `95` or sets `commands.test.value` to a custom script and re-runs `/codebase-analyzer`. The analyzer keeps the user's values untouched, refreshes `evidence` and `baseline`, and removes resolved items from `missing`.
- **Zero existing tests in repo:** `exemplarTestFiles.value` is `[]`, `baseline` reports 0 tests run, and a non-blocking `missing` item notes that standard framework idioms will be used.
- **Ambiguous test runner (e.g., both Jest and Vitest in `package.json`):** Record the most evidenced runner with `confidence: "low"` or `null` and add a `blocking: true` question in `missing`.
- **Monorepo:** Populate `monorepoSubprojects.value` with detected workspace paths (e.g. `["packages/core", "packages/api"]`).
- **Never modify source or test files:** The analyzer is strictly read-only outside `.unit-test-agent/project.json`.

---

## 6. Acceptance Criteria Checklist

- [ ] Creates `.unit-test-agent/project.json` conforming to `schemas/project.schema.json`.
- [ ] Every detected field contains `value`, `confidence` (`high` | `medium` | `low`), and `evidence`.
- [ ] Executes the detected test command and populates `baseline` with accurate pass/fail counts and `alreadyFailing` flag.
- [ ] Populates `missing` with `blocking: true` when `commands.test`, `testFramework`, `sourceRoots`, or `testRoots` cannot be determined.
- [ ] Populates `missing` with `blocking: false` when optional tools like `mutationTool` are absent.
- [ ] Re-running `/codebase-analyzer` after manual user edits to `project.json` preserves all non-null user-edited values and clears resolved `missing` entries.
- [ ] Modifies zero files outside `.unit-test-agent/project.json`.
