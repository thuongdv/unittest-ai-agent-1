# F14: Run Reporter (`unit-test-reporter`)

## 1. Purpose

Turn the JSON outputs of one run into a single human-readable `report.md` for developers, QA engineers, and BA/PO: one recommendation, a scorecard, the defects to fix, the requirement traceability matrix, and the open questions. The report is plain Markdown, so CI can post it to a pull request as-is.

The reporter **renders, it does not judge**. Every status, count, and ID in the report comes from a JSON file in the run. It never runs tests or tools.

---

## 2. Invocation and Arguments

| Runtime            | Interactive Invocation          | Headless / CLI Invocation                                    |
| :----------------- | :------------------------------ | :----------------------------------------------------------- |
| **Claude Code**    | `/unit-test-reporter <run-id>`  | `claude -p "/unit-test-reporter 20261105-101500-shop-142"`   |
| **Codex CLI**      | `$unit-test-reporter <run-id>`  | `codex exec "$unit-test-reporter 20261105-101500-shop-142"`  |
| **Gemini CLI**     | `/unit-test-reporter <run-id>`  | `gemini -p "/unit-test-reporter 20261105-101500-shop-142"`   |
| **GitHub Copilot** | `/unit-test-reporter <run-id>`  | N/A (VS Code Chat Agent mode)                                |

The orchestrator ([F12](F12-orchestrator.md)) runs it as the last step of both workflows.

---

## 3. Inputs and Outputs

### Preconditions & Inputs

1. `.unit-test-agent/project.json` exists, has a supported `schemaVersion`, and has zero `blocking: true` items in `missing`.
2. `.unit-test-agent/runs/<run-id>/` contains at least one of `generation.json`, `failure-analysis.json` with a supported `schemaVersion`.
3. Reads whichever of these exist: `requirements.json`, `generation.json` and the latest `generation.r<N>.json`, `review.json`, `failure-analysis.json`, `failure-repair.json`, plus `project.json` (`commands`, `policy.coverageThresholds`).

### Output

- **Path:** `.unit-test-agent/runs/<run-id>/report.md` (overwritten on every run). This is the only file the reporter writes.

### Rules

| Rule                  | Requirement                                                                                                      |
| :-------------------- | :--------------------------------------------------------------------------------------------------------------- |
| Render only           | Every status, number, and ID comes from a JSON field, or a count or join of JSON fields. Add no new findings.     |
| Latest data           | Use the latest `generation.r<N>.json` when present, else `generation.json`.                                      |
| Missing inputs        | Omit a section whose source file does not exist. Write `None.` for a section whose source list is empty.         |
| No side effects       | Never run tests or tools, edit other files, or contact Jira or a PR.                                             |
| No requirements       | When `requirements.json` is absent or `requirementCompliance.status` is `not_assessed`, show the banner: _"Requirements: none. Tests were derived from the implementation, so they confirm what the code does, not what the business asked for."_ |

### Recommendation Table

Evaluate every row. The first row that matches is the headline recommendation; every row that matches becomes a numbered item under **Actions**.

| # | Condition (from JSON)                                                                                         | Recommendation                                        | Owner          |
| - | :------------------------------------------------------------------------------------------------------------ | :---------------------------------------------------- | :------------- |
| 1 | Any `suspectedDefects` in the latest generation file or `failure-repair.json`                                | Do not merge. Fix production code (list defect IDs).  | Developer      |
| 2 | `requirements.json` has any `unclear` item                                                                    | Clarify requirements before merge (list UNC IDs + sources). | BA / PO   |
| 3 | `review.json` verdict `changes_requested`, any test case `failed`, or any `failure-repair.json` resolution `unresolved` / `blocked_env` | Tests need follow-up (list finding / failure IDs).    | Developer / QA |
| 4 | Any requirement classified `not_covered`                                                                      | Check where these requirements are implemented.       | QA             |
| 5 | Requirement compliance not assessed                                                                           | Approve tests. Verified against the implementation only. | Developer   |
| 6 | Otherwise                                                                                                     | Approve.                                              | —              |

### Report Sections (in order)

| Section                    | Content                                                                                                       | Source                                         |
| :------------------------- | :------------------------------------------------------------------------------------------------------------ | :--------------------------------------------- |
| Title & recommendation     | Run ID, headline recommendation, owner, run time, targets, requirement sources, workflow, review rounds used  | All files                                      |
| Summary                    | Requirement compliance counts, scenario coverage, open questions, test status counts, code coverage vs thresholds, mutation score, flakiness, review verdict | `review.json`, generation, `requirements.json` |
| Actions                    | Every matching row of the recommendation table                                                                | Recommendation table                           |
| For developers             | Defects to fix (location, requirement/scenario, expected, actual, test to re-enable), open review findings, uncovered lines with reasons | generation, `review.json`              |
| For QA                     | Requirement → scenario → test case → status → classification matrix; count of implementation-only tests; questions for BA/PO; surviving mutants | `requirements.json`, generation, `review.json` |
| Failure triage             | Failures by category, resolution status, environment steps the user must take                               | `failure-analysis.json`, `failure-repair.json` |
| Reproduce                  | Copy-paste commands to re-run the tests and coverage                                                          | `commandExecuted` fields, `project.json`       |

### Complete Example (`.unit-test-agent/runs/20261105-101500-shop-142/report.md`)

````markdown
# Unit Test Report · 20261105-101500-shop-142

> **Recommendation: Do not merge. Fix production code (DEF-001, DEF-002).** Owner: Developer

| Run                  | Targets           | Requirements    | Workflow                            |
| :------------------- | :---------------- | :-------------- | :---------------------------------- |
| 2026-11-05 10:42 UTC | `src/discount.ts` | SHOP-142 (Jira) | Generate & review · 2 review rounds |

## Summary

| Check                  | Result                                                         |
| :--------------------- | :------------------------------------------------------------- |
| Requirement compliance | 2 requirements: 1 BUG · 1 PASS · 0 TEST ISSUE · 0 UNCLEAR · 0 NOT COVERED |
| Scenario coverage      | 5 / 5 scenarios have a test                                    |
| Open questions         | 1 (UNC-001)                                                    |
| Tests                  | 4 passed · 2 skipped as defects · 0 failed                     |
| Code coverage          | lines 100% (min 80) · branches 100% (min 70)                   |
| Mutation score         | 88.9% (8 / 9 killed)                                           |
| Flakiness              | 3 / 3 runs passed                                              |
| Review                 | approved (round 2 of 3)                                        |

## Actions

1. **Developer:** fix DEF-001 and DEF-002, then re-enable TC-002 and TC-003.
2. **BA / PO:** answer UNC-001 in SHOP-142.

## For developers

### Defects to fix

| Defect  | Location             | Requirement       | Expected                  | Actual    | Re-enable after fix               |
| :------ | :------------------- | :---------------- | :------------------------ | :-------- | :-------------------------------- |
| DEF-001 | `src/discount.ts:12` | REQ-001 / SCN-002 | discount is $10 at $100   | returns 0 | TC-002 (`tests/discount.test.ts`) |
| DEF-002 | `src/discount.ts:12` | REQ-001 / SCN-003 | discount is $20 at $200   | returns 0 | TC-003 (`tests/discount.test.ts`) |

### Open test findings

None.

### Uncovered lines

None.

## For QA

### Requirement traceability

| Requirement | Scenario              | Test case | Test status        | Result |
| :---------- | :-------------------- | :-------- | :----------------- | :----- |
| REQ-001     | SCN-001 $99.99 → $0   | TC-001    | passed             | BUG    |
|             | SCN-002 $100 → $10    | TC-002    | skipped (DEF-001)  |        |
|             | SCN-003 $200 → $20    | TC-003    | skipped (DEF-002)  |        |
|             | SCN-004 $0 → $0       | TC-004    | passed             |        |
| REQ-002     | SCN-005 -$1 → error   | TC-005    | passed             | PASS   |

Tests derived from the implementation only (no requirement link): 1 (TC-006).

### Questions for BA / PO

| ID      | Source             | Question                                                           |
| :------ | :----------------- | :----------------------------------------------------------------- |
| UNC-001 | SHOP-142 (REQ-001) | Is the discount rounded to cents, and if so, half-up or half-even? |

### Surviving mutants

| Location             | Mutation                         | Note                                        |
| :------------------- | :------------------------------- | :------------------------------------------ |
| `src/discount.ts:12` | `amount > 200` → `amount >= 200` | On the DEF-001 line; re-check after the fix. |

## Reproduce

```bash
npx vitest run tests/discount.test.ts --coverage
```
````

---

## 4. Step-by-Step Behavior

1. **Check Preconditions** (Section 3).
2. **Load Run Files:** Read every available input; pick the latest generation file.
3. **Compute the Recommendation:** Apply the recommendation table.
4. **Render Sections:** Fill the sections in the order above. Show requirement-derived data only when `requirements.json` exists; otherwise show the no-requirements banner under the title.
5. **Write Output:** Write `report.md` and print its path and the headline recommendation.

---

## 5. Edge Cases

- **Failure workflow run (no `generation.json`):** Render Title, Summary (failure counts), Actions, Failure triage, and Reproduce only.
- **Review never ran (generator only):** Show `Review: not run`; recommendation row 3 cannot match on the verdict.
- **Round cap reached without approval:** Row 3 matches; list the outstanding finding IDs.
- **Re-run after a fix pass:** Overwrite `report.md` from the latest files.

---

## 6. Acceptance Criteria Checklist

- [ ] Writes only `.unit-test-agent/runs/<run-id>/report.md`; `git status` shows no other change.
- [ ] Headline recommendation and Actions match the recommendation table applied to the JSON files.
- [ ] Every defect ID, finding ID, requirement ID, and count in the report exists in, or is computed from, the run's JSON files.
- [ ] Shows the no-requirements banner whenever requirement compliance was not assessed.
- [ ] Contains the sections Summary, Actions, For developers, For QA (generate workflow) or Failure triage (failure workflow), and Reproduce.
