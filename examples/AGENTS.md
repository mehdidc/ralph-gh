# Agent instructions

## Setup
```bash
uv sync --dev --locked
```

## Validation
```bash
uv run ruff check .
uv run pytest -q
```

## Rules
- Work only on the assigned issue.
- Add regression tests for bugs.
- Avoid unrelated refactoring.
- Never push or merge; the outer runner owns GitHub operations.
