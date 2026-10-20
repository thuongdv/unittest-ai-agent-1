# Implementation Plan: F08 — Multi-Runtime Packaging and Installation (`runtimes/`)

## 1. Dependencies on Other Features

- **Depends on:**
  - [ADR-001: Packaging Architecture](../decisions/ADR-001-packaging.md)
  - Canonical agent files in `agents/` ([F01](F01-plan.md)–[F06](F06-plan.md))

## 2. Files to Create or Change

- **Claude Code (`runtimes/claude/.claude/`):**
  - `commands/codebase-analyzer.md`
  - `commands/unit-test-generator.md`
  - `commands/unit-test-reviewer.md`
  - `commands/unit-test-failure-analyzer.md`
  - `agents/unit-test-reviewer.md`
- **OpenAI Codex CLI (`runtimes/codex/`):**
  - `.agents/skills/codebase-analyzer/SKILL.md`
  - `.agents/skills/unit-test-generator/SKILL.md`
  - `.agents/skills/unit-test-reviewer/SKILL.md`
  - `.agents/skills/unit-test-failure-analyzer/SKILL.md`
  - `.codex/agents/unit-test-reviewer.md`
- **Gemini CLI (`runtimes/gemini/.gemini/`):**
  - `commands/codebase-analyzer.toml`
  - `commands/unit-test-generator.toml`
  - `commands/unit-test-reviewer.toml`
  - `commands/unit-test-failure-analyzer.toml`
  - `agents/unit-test-reviewer.md`
- **GitHub Copilot (`runtimes/copilot/.github/`):**
  - `prompts/codebase-analyzer.prompt.md`
  - `prompts/unit-test-generator.prompt.md`
  - `prompts/unit-test-reviewer.prompt.md`
  - `prompts/unit-test-failure-analyzer.prompt.md`
  - `agents/unit-test-reviewer.agent.md`

## 3. Ordered Tasks

### Task F08-T1: Create Thin Wrappers & Reviewer Subagent for Claude Code and Codex CLI

- **Description:** Create the thin wrapper files under `runtimes/claude/.claude/` and `runtimes/codex/{.agents,.codex}/` pointing to `.unit-test-agent/agents/<agent>.md`.
- **Acceptance Criteria:**
  - [ ] Zero duplicated business rules; each wrapper is 5–15 lines.
  - [ ] Reviewer wrapper delegates to the isolated `unit-test-reviewer` subagent.
- **Files Touched:** `runtimes/claude/.claude/**`, `runtimes/codex/**`

### Task F08-T2: Create Thin Wrappers & Reviewer Subagent for Gemini CLI and GitHub Copilot

- **Description:** Create `.toml` commands and reviewer subagent under `runtimes/gemini/.gemini/` and `.prompt.md` / `.agent.md` wrappers under `runtimes/copilot/.github/`.
- **Acceptance Criteria:**
  - [ ] Gemini `.toml` files use `@{.unit-test-agent/agents/<agent>.md}` and `{{args}}`.
  - [ ] Copilot `.prompt.md` files use `mode: agent` and `agent/runSubagent` for the reviewer.
- **Files Touched:** `runtimes/gemini/.gemini/**`, `runtimes/copilot/.github/**`

## 4. How to Verify Manually

1. Copy `agents/`, `schemas/`, and each runtime directory into a temp fixture directory via `cp -r`.
2. Confirm all file references in the wrappers resolve to `.unit-test-agent/agents/<agent>.md`.
