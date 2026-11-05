# Implementation Plan: F09 — Example Fixture Projects and Manual Test Procedure (`examples/`)

## 1. Dependencies on Other Features

- **Depends on:** None (Can be built right after [F07: JSON Contracts](F07-plan.md) so fixtures are ready for testing F01–F06 and F11).

## 2. Files to Create or Change

- **TypeScript + Vitest Fixture (`examples/ts-vitest/`):**
  - `package.json`, `tsconfig.json`, `vitest.config.ts`
  - `src/cart.ts`, `src/pricing.ts`, `src/order.ts`, `src/discount.ts`
  - `requirements/SHOP-142.md`
  - `tests/cart.test.ts`, `tests/regression/order.failing.ts`
  - `sample-failure.log`
- **Python + pytest Fixture (`examples/py-pytest/`):**
  - `pyproject.toml`
  - `src/inventory/__init__.py`, `src/inventory/stock.py`, `src/inventory/reorder.py`, `src/inventory/sku.py`, `src/inventory/restock.py`
  - `requirements/INV-77.md`
  - `tests/test_stock.py`, `tests/regression/test_sku_failing.py`
  - `sample-failure.log`
- **Manual Test Procedure:**
  - `docs/manual-test-procedure.md`

## 3. Ordered Tasks

### Task F09-T1: Create `examples/ts-vitest/` Fixture Project

- **Description:** Build the TypeScript + Vitest fixture with `cart.ts` + `tests/cart.test.ts` (passing exemplar), `pricing.ts` (branching logic + deliberate discount bug + unreachable defensive branch), `order.ts` (updated `ORD-<year>-<id>` format), `discount.ts` + `requirements/SHOP-142.md` (code and JSDoc agree, requirement says $100 not $200; no rounding rule), `tests/regression/order.failing.ts` (1 `stale_test` + 1 `source_regression_bug`), and `sample-failure.log`.
- **Acceptance Criteria:**
  - [ ] `npm test` (`npx vitest run`) passes baseline `tests/cart.test.ts`.
  - [ ] `npx vitest run tests/regression/order.failing.ts` fails on the 2 seeded failures.
  - [ ] `discount.ts`'s JSDoc matches its code; only `SHOP-142.md` contradicts it.
- **Files Touched:** `examples/ts-vitest/**`

### Task F09-T2: Create `examples/py-pytest/` Fixture Project & `docs/manual-test-procedure.md`

- **Description:** Build the Python + pytest fixture with `stock.py` + `tests/test_stock.py` (passing exemplar), `reorder.py` (branching logic + deliberate boundary/arithmetic bug), `sku.py` (updated SKU format), `restock.py` + `requirements/INV-77.md` (code and docstring agree, requirement says ≤ 20% not < 10%; negative stock unstated), `tests/regression/test_sku_failing.py` (1 `stale_test` + 1 `source_regression_bug`), `sample-failure.log`, and document the manual test procedure in `docs/manual-test-procedure.md`.
- **Acceptance Criteria:**
  - [ ] `pytest` passes baseline `tests/test_stock.py`.
  - [ ] `pytest tests/regression/test_sku_failing.py` fails on the 2 seeded failures.
  - [ ] `restock.py`'s docstring matches its code; only `INV-77.md` contradicts it.
  - [ ] `docs/manual-test-procedure.md` provides step-by-step copy-paste verification commands.
- **Files Touched:** `examples/py-pytest/**`, `docs/manual-test-procedure.md`

## 4. How to Verify Manually

1. Run `npm install && npx vitest run` in a temp copy of `examples/ts-vitest`.
2. Run `pytest` in a temp copy of `examples/py-pytest`.
