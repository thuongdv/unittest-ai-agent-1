# Repository Instructions for AI Coding Agents (`AGENTS.md`)

## 1. Session Bootstrap Protocol (READ FIRST)

Every session working on this repository **MUST** begin with these steps:

1. Read [`PROGRESS.md`](PROGRESS.md) to check the current project phase, the **Master Task Table**, **Open Questions**, and any `blocked` items.
2. Read [`docs/prompt.md`](docs/prompt.md) and [`docs/decisions/ADR-001-packaging.md`](docs/decisions/ADR-001-packaging.md) for architectural context.
3. Resume work from the next `todo` task in [`PROGRESS.md`](PROGRESS.md) in dependency order (only after human approval of Step 2 specs/plans).
4. Before implementing any feature `FXX`, read both its specification (`docs/features/FXX-*.md`) and its plan (`docs/plans/FXX-plan.md`).

---

## 2. Hard Project Constraints

- **No helper code in the agents:** Deliverables installed into user repositories are strictly `.md` instruction files, `.json` / `.schema.json` files, and the thinnest runtime wrapper files (`.toml` for Gemini CLI, `.prompt.md` / `.agent.md` for Copilot, `SKILL.md` for Codex, `.md` for Claude Code). Never write `.js`, `.ts`, `.py`, or `.sh` scripts for the agents themselves.
  - _Exception:_ Developer evaluation tooling in `evals/` (`F11`) and test fixtures in `examples/` (`F09`) contain TypeScript/Python code used exclusively to test this repository.
- **Canonical single source of truth:** All agent logic lives in `agents/<agent-name>.md`. Files under `runtimes/<runtime>/` must remain thin wrappers (5–15 lines) with zero duplicated rules.
- **Concise, imperative agent instructions:** Write agent prompts in `agents/*.md` using short, imperative rules and structured markdown tables. Avoid long prose.
- **Ambiguity rule:** If a requirement is ambiguous, do not guess—log the question under **Open Questions** in [`PROGRESS.md`](PROGRESS.md) and ask the user.

---

## 3. Development & Task Completion Workflow

1. Work one task at a time from [`PROGRESS.md`](PROGRESS.md).
2. Mark the task `in-progress` in [`PROGRESS.md`](PROGRESS.md) when starting.
3. Verify the task against all acceptance criteria in `docs/features/FXX-*.md` and `docs/plans/FXX-plan.md`.
4. Mark the task `done` in [`PROGRESS.md`](PROGRESS.md), append an entry to the dated **Changelog**, and commit using a Conventional Commit message (e.g., `feat(schemas): add project, generation, and review JSON schemas`).
