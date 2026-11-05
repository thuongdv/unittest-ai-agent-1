# F08: Multi-Runtime Packaging and Installation (`agents/`, `runtimes/`)

## 1. Purpose

Package the six unit testing agents (`codebase-analyzer`, `unit-test-generator`, `unit-test-reviewer`, `unit-test-failure-analyzer`, `unit-test-requirement-analyzer` ([F13](F13-requirement-analyzer.md)), `unit-test-reporter` ([F14](F14-reporter.md))) and the `unit-test-orchestrator` ([F12](F12-orchestrator.md)) so they can be installed into any user repository via plain file copying (`cp -r`) and invoked natively in **Claude Code**, **OpenAI Codex CLI**, **Gemini CLI**, and **GitHub Copilot (VS Code)** without duplicating canonical prompt instructions or using helper scripts.

---

## 2. Invocation and Arguments

Follows the packaging architecture selected in [ADR-001](../decisions/ADR-001-packaging.md) and documented in [runtime-compatibility.md](../runtime-compatibility.md).

### Copy-Based Installation Commands (in Target Repo)

```bash
# 1. Copy canonical agent instructions, JSON schemas, and signature extractors (required for all runtimes)
mkdir -p <target-repo>/.unit-test-agent
cp -r agents schemas tools <target-repo>/.unit-test-agent/

# 2. Copy the thin wrapper for your runtime(s):
# Claude Code:
cp -r runtimes/claude/.claude <target-repo>/
# OpenAI Codex CLI:
cp -r runtimes/codex/.agents runtimes/codex/.codex <target-repo>/
# Gemini CLI:
cp -r runtimes/gemini/.gemini <target-repo>/
# GitHub Copilot (VS Code):
cp -r runtimes/copilot/.github <target-repo>/
```

---

## 3. Inputs and Outputs

### Repository Layout Delivered by F08

```text
tools/                       # read-only signature extractors (F13, DEC-007)
├── extract-signatures.mjs
└── extract_signatures.py

agents/
├── codebase-analyzer.md
├── unit-test-generator.md
├── unit-test-reviewer.md
├── unit-test-failure-analyzer.md
├── unit-test-requirement-analyzer.md
├── unit-test-reporter.md
└── unit-test-orchestrator.md

runtimes/
├── claude/
│   └── .claude/
│       ├── commands/
│       │   ├── codebase-analyzer.md
│       │   ├── unit-test-generator.md
│       │   ├── unit-test-reviewer.md
│       │   ├── unit-test-failure-analyzer.md
│       │   ├── unit-test-requirement-analyzer.md
│       │   ├── unit-test-reporter.md
│       │   └── unit-test-orchestrator.md
│       └── agents/
│           ├── unit-test-reviewer.md
│           └── unit-test-requirement-analyzer.md
├── codex/
│   ├── .agents/
│   │   └── skills/
│   │       ├── codebase-analyzer/SKILL.md
│   │       ├── unit-test-generator/SKILL.md
│   │       ├── unit-test-reviewer/SKILL.md
│   │       ├── unit-test-failure-analyzer/SKILL.md
│   │       ├── unit-test-requirement-analyzer/SKILL.md
│   │       ├── unit-test-reporter/SKILL.md
│   │       └── unit-test-orchestrator/SKILL.md
│   └── .codex/
│       └── agents/
│           ├── unit-test-reviewer.md
│           └── unit-test-requirement-analyzer.md
├── gemini/
│   └── .gemini/
│       ├── commands/
│       │   ├── codebase-analyzer.toml
│       │   ├── unit-test-generator.toml
│       │   ├── unit-test-reviewer.toml
│       │   ├── unit-test-failure-analyzer.toml
│       │   ├── unit-test-requirement-analyzer.toml
│       │   ├── unit-test-reporter.toml
│       │   └── unit-test-orchestrator.toml
│       └── agents/
│           ├── unit-test-reviewer.md
│           └── unit-test-requirement-analyzer.md
└── copilot/
    └── .github/
        ├── prompts/
        │   ├── codebase-analyzer.prompt.md
        │   ├── unit-test-generator.prompt.md
        │   ├── unit-test-reviewer.prompt.md
        │   ├── unit-test-failure-analyzer.prompt.md
        │   ├── unit-test-requirement-analyzer.prompt.md
        │   ├── unit-test-reporter.prompt.md
        │   └── unit-test-orchestrator.prompt.md
        └── agents/
            ├── unit-test-reviewer.agent.md
            └── unit-test-requirement-analyzer.agent.md
```

### Wrapper Examples

#### 1. Claude Code Wrapper (`runtimes/claude/.claude/commands/unit-test-generator.md`)

```markdown
---
description: Generate unit tests, fix reviewer findings (--fix <run>), or repair failing tests (--repair-failures <run>).
argument-hint: <targets> | --fix <run-id> | --repair-failures <run-id>
---

Follow the canonical agent instructions in @.unit-test-agent/agents/unit-test-generator.md.

User arguments: `$ARGUMENTS`
```

#### 2. Claude Code Reviewer Command + Subagent (`runtimes/claude/.claude/commands/unit-test-reviewer.md`)

```markdown
---
description: Audit generated unit tests in a fresh subagent context (rubric + mutation testing + 3x flakiness rerun).
argument-hint: <run-id>
---

Delegate this review to the `unit-test-reviewer` subagent (`.claude/agents/unit-test-reviewer.md`) in a fresh, isolated context window. Pass the run ID: `$ARGUMENTS`.
```

#### 3. Gemini CLI Wrapper (`runtimes/gemini/.gemini/commands/unit-test-generator.toml`)

```toml
description = "Generate unit tests, fix reviewer findings (--fix <run>), or repair failing tests (--repair-failures <run>)."
prompt = """
@{.unit-test-agent/agents/unit-test-generator.md}

User arguments: {{args}}
"""
```

#### 4. Codex CLI Skill Wrapper (`runtimes/codex/.agents/skills/unit-test-generator/SKILL.md`)

```markdown
---
name: unit-test-generator
description: Generate unit tests for target files, apply review fixes (--fix <run>), or repair failing regression tests (--repair-failures <run>).
---

Read and strictly follow `.unit-test-agent/agents/unit-test-generator.md` using the arguments provided by the user.
```

#### 5. GitHub Copilot Prompt Wrapper (`runtimes/copilot/.github/prompts/unit-test-reviewer.prompt.md`)

```markdown
---
mode: agent
description: Review generated unit tests in an isolated subagent context.
tools: ["agent/runSubagent", "read_file", "edit_file", "run_in_terminal"]
---

Use `runSubagent` with `agentName: "unit-test-reviewer"` to execute the instructions in [unit-test-reviewer.md](../../.unit-test-agent/agents/unit-test-reviewer.md) in a fresh context for the specified `<run-id>`.
```

---

## 4. Step-by-Step Behavior

1. **Canonical Resolution:** Every wrapper points to `.unit-test-agent/agents/<agent-name>.md`.
2. **Argument Forwarding:** Every wrapper passes the raw user argument string (`<targets>`, `--fix <run-id>`, `--repair-failures <run-id>`, or `--log <file>`) directly to the canonical agent.
3. **Subagent Context Isolation for Reviewer and Requirement Analyzer:** Every runtime wrapper for `unit-test-reviewer` and `unit-test-requirement-analyzer` invokes its respective isolated subagent definition, so the reviewer never shares conversation history with the generator and the requirement analyzer never shares context with any step that read source code.
4. **Orchestrator Wrapper:** Every runtime wrapper for `unit-test-orchestrator` forwards arguments to `.unit-test-agent/agents/unit-test-orchestrator.md` and grants the runtime's subagent tool (Claude Code `Task`, Codex subagents, Gemini `@<agent>`, Copilot `agent/runSubagent`) so the orchestrator can delegate the reviewer and the requirement analyzer to their dedicated subagents. No other subagent definitions are added.
5. **Jira MCP Access:** Wrappers do not configure MCP servers; the user configures a Jira MCP server in their runtime (README, [F10](F10-readme-user-guide.md)). The requirement-analyzer subagent (and the orchestrator, where tools are listed explicitly, e.g., Copilot `tools:`) must be granted that server's read tools and the shell (to run `.unit-test-agent/tools/` extractors). Where a runtime supports per-subagent tool lists, grant only Jira read tools, the shell, and file read/write.

---

## 5. Edge Cases

- **Target repo already has `.claude/`, `.gemini/`, or `.github/`:** `cp -r` merges the `commands/`, `prompts/`, `skills/`, and `agents/` subdirectories into the existing dot-directories without overwriting unrelated project files (such as `.github/workflows/`).
- **Jira MCP server not configured:** The requirement analyzer stops with a pointer to the README setup section; other agents are unaffected.
- **User forgets to copy `agents/` and `schemas/` into `.unit-test-agent/`:** Wrappers instruct the agent to stop immediately and tell the user to copy `agents/` and `schemas/` into `.unit-test-agent/`.

---

## 6. Acceptance Criteria Checklist

- [ ] All 7 canonical instruction files (6 agents + orchestrator) live in `agents/` with zero duplicated business rules in `runtimes/`.
- [ ] Thin wrappers exist for all 6 agents and the orchestrator across all 4 runtimes (`runtimes/claude/`, `runtimes/codex/`, `runtimes/gemini/`, `runtimes/copilot/`).
- [ ] Each runtime includes isolated subagent configurations for `unit-test-reviewer` and `unit-test-requirement-analyzer`.
- [ ] The requirement analyzer can read a Jira issue through the user's Jira MCP server in each runtime.
- [ ] Contains zero `.js`, `.ts`, `.py`, or `.sh` scripts outside `tools/` (the read-only signature extractors).
- [ ] Installing into a fixture project requires only `cp -r` commands.
