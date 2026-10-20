# Runtime Compatibility Matrix (Claude Code, OpenAI Codex CLI, Gemini CLI, GitHub Copilot)

_Research Date: 2026-10-20_

## Overview

This document compares how the four target AI coding runtimes handle custom commands/prompts, agent skills (`SKILL.md`), isolated subagents, argument passing, and headless execution. It serves as the foundation for [ADR-001: Multi-Runtime Packaging Architecture](decisions/ADR-001-packaging.md).

---

## 1. Compatibility Matrix

| Dimension                                 | Claude Code                                                                                                                        | OpenAI Codex CLI                                                                                             | Gemini CLI                                                                                                                         | GitHub Copilot (VS Code)                                                                                |
| :---------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **Project-Level Commands Path**           | `.claude/commands/<name>.md` (legacy)<br>`.claude/skills/<name>/SKILL.md` (modern)                                                 | `.agents/skills/<name>/SKILL.md` (standard)<br>`.codex/prompts/<name>.md` (legacy)                           | `.gemini/commands/<name>.toml`                                                                                                     | `.github/prompts/<name>.prompt.md`                                                                      |
| **Project-Level Subagents Path**          | `.claude/agents/<name>.md`                                                                                                         | `.codex/agents/<name>.toml` or `.md`                                                                         | `.gemini/agents/<name>.md`                                                                                                         | `.github/agents/<name>.agent.md`                                                                        |
| **Agent Skills (`SKILL.md`) Support**     | **Yes** (`.claude/skills/<name>/SKILL.md`)                                                                                         | **Yes** (`.agents/skills/<name>/SKILL.md`)                                                                   | **Yes** (`.gemini/skills/<name>/SKILL.md` & `.agents/skills/<name>/SKILL.md`)                                                      | **Yes** (`.github/skills/<name>/SKILL.md`)                                                              |
| **Wrapper File Format & Frontmatter**     | Markdown with YAML frontmatter (`name`, `description`, `argument-hint`, `allowed-tools`)                                           | Markdown (`SKILL.md`) with YAML frontmatter (`name`, `description`)                                          | TOML (`description`, `prompt`) for `/commands`; Markdown + YAML frontmatter for skills/subagents                                   | Markdown (`*.prompt.md` / `*.agent.md`) with YAML frontmatter (`mode: 'agent'`, `description`, `tools`) |
| **How Arguments Are Passed**              | `$ARGUMENTS` (full string) or positional `$1`, `$2` inside command/skill body                                                      | Passed as user prompt context following skill invocation (`$<skill-name> <args>`) or `$ARGUMENTS` in prompts | `{{args}}` placeholder inside `.gemini/commands/<name>.toml` `prompt` string                                                       | Freeform text typed after `/<prompt-name> <args>` in Chat input (or `${input:var}` variables)           |
| **File Inclusion / Reference Syntax**     | `@path/to/file.md` (inlines/references file automatically)                                                                         | Agent reads referenced path (`agents/<name>.md`) or loads `SKILL.md` progressively                           | `@{path/to/file.md}` inside `.toml` prompt templates; `@path/to/file.md` in interactive chat                                       | `[file](path/to/file.md)` or `#file:path/to/file.md` Markdown link reference in prompt/agent files      |
| **Fresh-Context Subagent Support**        | **Yes** — via `Task` / subagent delegation to `.claude/agents/<name>.md`                                                           | **Yes** — via subagent spawning (`/subagents` / `.codex/agents/`) or fresh `codex exec` invocation           | **Yes** — via `.gemini/agents/<name>.md` (`@<agent-name>`) running in an isolated context window                                   | **Yes** — via `agent/runSubagent` tool or selecting `.github/agents/<name>.agent.md` in a new chat      |
| **Interactive User Invocation**           | `/codebase-analyzer`<br>`/unit-test-generator <targets>`<br>`/unit-test-reviewer <run>`<br>`/unit-test-failure-analyzer [targets]` | `$codebase-analyzer` (or `/skills` menu) or `/prompts:codebase-analyzer`                                     | `/codebase-analyzer`<br>`/unit-test-generator <targets>`<br>`/unit-test-reviewer <run>`<br>`/unit-test-failure-analyzer [targets]` | `/codebase-analyzer` in Copilot Chat (Agent mode)                                                       |
| **Headless / CLI Invocation (for Evals)** | `claude -p "/unit-test-generator src/foo.ts"`                                                                                      | `codex exec "$unit-test-generator src/foo.ts"`                                                               | `gemini -p "/unit-test-generator src/foo.ts"`                                                                                      | Manual in VS Code UI (or GitHub Copilot CLI)                                                            |

---

## 2. Detailed Runtime Breakdown

### 2.1 Claude Code

- **Commands & Skills:** Supports both `.claude/commands/<name>.md` and `.claude/skills/<name>/SKILL.md`. Both expose `/<name>` as a user-invokable slash command.
- **Arguments:** `$ARGUMENTS` expands to whatever arguments the user passed after `/<name>`.
- **Subagents (Fresh Context):** `.claude/agents/<name>.md` defines an isolated subagent with its own context window, system prompt, and `tools` list. When `/unit-test-reviewer <run>` is invoked, the command wrapper can instruct Claude Code to delegate the review to the `unit-test-reviewer` subagent so generator bias is completely eliminated.
- **Official References:**
  - Claude Code Slash Commands & Skills: `https://docs.anthropic.com/en/docs/claude-code/slash-commands`
  - Claude Code Subagents: `https://docs.anthropic.com/en/docs/claude-code/sub-agents`
  - Open Agent Skills Specification: `https://agentskills.io`

### 2.2 OpenAI Codex CLI

- **Skills & Prompts:** Codex natively supports the open Agent Skills specification at `.agents/skills/<name>/SKILL.md` (discoverable via `/skills` or `$<name>`) as well as custom prompts.
- **Arguments:** Any arguments typed after the skill or prompt invocation are passed directly in the user request turn.
- **Subagents (Fresh Context):** Codex CLI supports multi-agent orchestration (`.codex/agents/`) and headless isolated execution via `codex exec`.
- **Official References:**
  - OpenAI Codex CLI Repository & Docs: `https://github.com/openai/codex`
  - AGENTS.md Open Standard: `https://agents.md`

### 2.3 Gemini CLI

- **Custom Commands:** Defined as TOML files in `.gemini/commands/<name>.toml`.
  - Uses `{{args}}` to inject command-line arguments.
  - Supports native file injection via `@{.unit-test-agent/agents/<name>.md}` directly inside the `prompt` field, allowing zero-duplication wrappers!
- **Skills & Subagents:** Supports `.gemini/skills/<name>/SKILL.md` (and `.agents/skills/`) and isolated subagents in `.gemini/agents/<name>.md` with YAML frontmatter.
- **Official References:**
  - Gemini CLI Custom Commands: `https://geminicli.com/docs/cli/commands/`
  - Gemini CLI Agent Skills: `https://geminicli.com/docs/cli/skills/`

### 2.4 GitHub Copilot (VS Code Agent Mode)

- **Prompt Files:** Stored in `.github/prompts/<name>.prompt.md` with YAML frontmatter (`mode: 'agent'`, `description`, `tools`). Invoked in Copilot Chat via `/<name> <args>`.
- **Custom Agents & Subagents:** Stored in `.github/agents/<name>.agent.md`. Supports isolated subagent invocation via the `agent/runSubagent` tool (`runSubagent` with `agentName`, `description`, `prompt`) or starting a fresh chat session with the reviewer agent.
- **Skills:** Supports `.github/skills/<name>/SKILL.md`.
- **Official References:**
  - VS Code Copilot Prompt Files & Custom Instructions: `https://code.visualstudio.com/docs/copilot/copilot-customization`
  - GitHub Copilot Custom Agents & Subagents: `https://docs.github.com/en/copilot/customizing-copilot`

---

## 3. Key Findings for Packaging

1. **Canonical Instructions:** All four runtimes can read Markdown files from the repository. Placing the 4 canonical agent instruction files in `agents/` (and installing them into `.unit-test-agent/agents/` in a target repository) ensures a single source of truth.
2. **Thin Per-Runtime Wrappers:**
   - **Claude Code:** `.claude/commands/<agent>.md` (plus `.claude/agents/unit-test-reviewer.md` for subagent isolation).
   - **Codex CLI:** `.agents/skills/<agent>/SKILL.md`.
   - **Gemini CLI:** `.gemini/commands/<agent>.toml` using `@{.unit-test-agent/agents/<agent>.md}` and `{{args}}` (plus `.gemini/agents/unit-test-reviewer.md`).
   - **GitHub Copilot:** `.github/prompts/<agent>.prompt.md` using `agent/runSubagent` for the reviewer (plus `.github/agents/unit-test-reviewer.agent.md`).
3. **Zero Duplication:** Every wrapper is 5–15 lines long and simply forwards the user's arguments to the canonical instruction file in `.unit-test-agent/agents/<agent>.md`.
