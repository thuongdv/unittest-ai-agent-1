# Claude Code Repository Instructions (`CLAUDE.md`)

@AGENTS.md

## Quick Reference for Claude Code Sessions

1. **Always start by reading [`PROGRESS.md`](PROGRESS.md).** Resume from the next `todo` task in the Master Task Table once Step 2 review is approved by the user.
2. **Specifications & Plans:**
   - Runtime research: [`docs/runtime-compatibility.md`](docs/runtime-compatibility.md)
   - Packaging ADR: [`docs/decisions/ADR-001-packaging.md`](docs/decisions/ADR-001-packaging.md)
   - Feature specs (`F01`–`F11`): `docs/features/`
   - Implementation plans (`F01`–`F11`): `docs/plans/`
3. **After every completed task:**
   - Update task status and changelog in [`PROGRESS.md`](PROGRESS.md).
   - Commit with a Conventional Commit message (`feat:`, `docs:`, `test:`, `chore:`).
