# F09: Example Fixture Projects and Manual Test Procedure (`examples/`)

## 1. Purpose

Provide two small, self-contained, realistic fixture repositories in `examples/`—one in **TypeScript + Vitest** (`examples/ts-vitest/`) and one in **Python + pytest** (`examples/py-pytest/`)—along with a repeatable manual verification procedure to test all four agents and the orchestrator end-to-end across all four runtimes.

---

## 2. Invocation and Arguments

Fixtures are never installed into a user's repository. To test manually or via `evals/` ([F11](F11-automated-output-checks.md)), copy a fixture to a temporary directory, initialize Git, copy the `.unit-test-agent/` and runtime wrapper files into it, and invoke the agents.

---

## 3. Inputs and Outputs

### Fixture 1: `examples/ts-vitest/` (TypeScript + Vitest)

```text
examples/ts-vitest/
├── package.json              # typescript, vitest, @vitest/coverage-v8
├── tsconfig.json
├── vitest.config.ts
├── src/
│   ├── cart.ts               # Existing tested module (used for style exemplar)
│   ├── pricing.ts            # Untested module with branching logic + 1 deliberate bug + 1 unreachable guard
│   └── order.ts              # Module with an intentional contract change (ORD-<year>-<id>)
└── tests/
    ├── cart.test.ts          # Passing exemplar test file
    └── regression/           # Optional/seeded failing tests for F05 & F06 testing
        └── order.failing.ts  # Stale test (expects old ORD-<id>) + regression test hitting pricing.ts bug
```

### Fixture 2: `examples/py-pytest/` (Python + pytest)

```text
examples/py-pytest/
├── pyproject.toml            # pytest, pytest-cov
├── src/
│   └── inventory/
│       ├── __init__.py
│       ├── stock.py          # Existing tested module (used for style exemplar)
│       ├── reorder.py        # Untested module with branching logic + 1 deliberate bug
│       └── sku.py            # Module with intentional format contract (SKU-<category>-<id>)
└── tests/
    ├── test_stock.py         # Passing exemplar test file
    └── regression/
        └── test_sku_failing.py # Stale test + regression test hitting reorder.py bug
```

### Required Characteristics in Each Fixture

1. **Passing Baseline + Style Exemplar:** By default (`npx vitest run` / `pytest`), `tests/cart.test.ts` and `tests/test_stock.py` pass and establish clear naming, assertion, and Arrange-Act-Assert conventions, while `tests/regression/` is excluded from the default test glob until activated for failure-analysis testing (or passed explicitly as a target/log).
2. **Branching Logic:** `pricing.ts` and `reorder.py` include multi-branch conditionals, boundary thresholds (`>= 100`, `=== 0`), error throwing (`RangeError` / `ValueError`), and an unreachable defensive guard.
3. **Deliberate Bug in Production Code:**
   - In `examples/ts-vitest/src/pricing.ts`: `applyDiscount(price, discountPercent)` has a documented JSDoc contract (`@returns price reduced by discountPercent, floored at 0`), but when `discountPercent === 100` (or in `calculateMemberPrice`), a deliberate arithmetic bug returns a wrong value.
   - In `examples/py-pytest/src/inventory/reorder.py`: `calculate_reorder_quantity(current, minimum, max_capacity)` has a docstring stating it returns `max_capacity - current` when `current <= minimum`, but the code uses `current < minimum` (off-by-one boundary bug) or adds instead of subtracts.
4. **Seeded Failing Tests for `F05` & `F06`:**
   - One **stale test** (`stale_test`) where the source function (`order.ts` / `sku.py`) clearly documents a newer return format, while the test asserts an older format.
   - One **regression test** (`source_regression_bug`) where the test asserts the mathematically/documented right value and fails because of the deliberate production bug in `pricing.ts` / `reorder.py`.
   - A pre-captured failure log (`examples/ts-vitest/sample-failure.log` and `examples/py-pytest/sample-failure.log`) to test `/unit-test-failure-analyzer --log <file>`.

---

## 4. Step-by-Step Manual Verification Procedure (`docs/manual-test-procedure.md` or inside `F09`)

1. **Prepare Sandbox Copy:**
   ```bash
   TMP_DIR=$(mktemp -d)
   cp -r examples/ts-vitest/. "$TMP_DIR/"
   mkdir -p "$TMP_DIR/.unit-test-agent"
   cp -r agents schemas "$TMP_DIR/.unit-test-agent/"
   cp -r runtimes/claude/.claude "$TMP_DIR/"
   cd "$TMP_DIR" && git init && npm install && git add . && git commit -m "chore: baseline"
   ```
2. **Step A — Test `codebase-analyzer` (F01):**
   - Run `/codebase-analyzer`.
   - Verify `.unit-test-agent/project.json` is created, `baseline.passed` > 0, and `missing` has no blocking items.
   - Edit `policy.coverageThresholds.lines` to `85`, re-run `/codebase-analyzer`, and verify `85` is preserved.
3. **Step B — Test `unit-test-generator` Generate Mode (F02):**
   - Run `/unit-test-generator src/pricing.ts`.
   - Verify `tests/pricing.test.ts` is created, deliberate bug is recorded in `suspectedDefects` with its test skipped/expected-fail, and `git diff src/` is empty.
4. **Step C — Test `unit-test-reviewer` (F04) & Fix Mode (F03):**
   - Run `/unit-test-reviewer <run-id>`.
   - Verify `review.json` is written and `flakinessCheck.reruns` is `3`.
   - If `verdict` is `"changes_requested"`, run `/unit-test-generator --fix <run-id>` and verify `generation.r1.json` is created.
5. **Step D — Test `unit-test-failure-analyzer` (F05) & Repair-Failures Mode (F06):**
   - Run `/unit-test-failure-analyzer tests/regression/order.failing.ts` (and test `--log sample-failure.log`).
   - Verify `failure-analysis.json` classifies the stale test as `stale_test` (`update_test`) and the pricing bug test as `source_regression_bug` (`skip_and_report_defect`).
   - Run `/unit-test-generator --repair-failures <failure-run-id>`.
   - Verify `failure-repair.json` shows the stale test fixed, the regression test marked skipped with `suspectedDefects` populated, and re-running the test file exits with code 0.
6. **Step E — Test `unit-test-orchestrator` (F12):**
   - Reset the sandbox to the baseline commit (keep `project.json`), run `/unit-test-orchestrator src/pricing.ts`, and verify Steps B–C outputs appear under one `<run-id>` with at most 2 `generation.r*.json` files.
   - Run `/unit-test-orchestrator --failures tests/regression/order.failing.ts` and verify Step D outputs appear under one `<run-id>`.
   - Delete `project.json`, run the orchestrator again, and verify it stops with a pointer to `/codebase-analyzer` and creates no `runs/` directory.

---

## 5. Edge Cases

- **Fixture isolation:** Fixtures must never be modified in-place during agent runs; always copy to a temporary directory first.
- **No network required after dependency install:** Once `npm install` or `pip install` has been run in the fixture environment, all tests and coverage commands run completely offline.

---

## 6. Acceptance Criteria Checklist

- [ ] `examples/ts-vitest/` and `examples/py-pytest/` exist and pass baseline tests out of the box.
- [ ] Both fixtures include branching logic, an unreachable defensive branch, at least one deliberate bug in production code, and seeded failing tests (`stale_test` + `source_regression_bug`) plus a sample `.log` file.
- [ ] Manual test procedure covers all 4 agents, all 3 generator modes (`generate`, `--fix`, `--repair-failures`), and both orchestrator workflows end-to-end.
