# Implementation Plan: F10 — README and User Guide (`README.md`)

## 1. Dependencies on Other Features

- **Depends on:** [F01](F01-plan.md)–[F09](F09-plan.md), [F11](F11-plan.md), and [F12](F12-plan.md).

## 2. Files to Create or Change

- `README.md`

## 3. Ordered Tasks

### Task F10-T1: Write Comprehensive `README.md` User & Contributor Guide

- **Description:** Replace the placeholder `README.md` with full documentation covering project overview, hard guardrails, copy-based installation for all 4 runtimes, one-time `/codebase-analyzer` setup, Workflow 1 (`/unit-test-orchestrator <targets>`: Generate → Review → Fix), Workflow 2 (`/unit-test-orchestrator --failures`: Failure Analyze → Repair Failures), the equivalent manual commands, `project.json` configuration & `missing` resolution, `suspectedDefects` workflow, and running `evals/`.
- **Acceptance Criteria:**
  - [ ] All copy-based installation commands and agent invocations match `F01`–`F12`.
  - [ ] All links to `docs/`, `schemas/`, `agents/`, `runtimes/`, `examples/`, and `evals/` are valid.
- **Files Touched:** `README.md`

## 4. How to Verify Manually

1. Follow the `cp -r` commands in `README.md` verbatim into a fresh temp directory and confirm all paths match the repository structure.
