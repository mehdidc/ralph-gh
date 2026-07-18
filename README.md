# ralph-gh

A minimal GitHub-native Ralph loop. GitHub issues replace PRD JSON files; labels form the queue/state machine; issue comments allow live human steering; fresh agent processes prevent context rot; Git commits and `.ralph/progress.md` retain memory.

## Flow

```text
ralph-ready issue
  -> isolated git worktree
  -> fresh agent iteration
  -> deterministic verification
  -> repeat if failing
  -> push branch
  -> draft PR + human review
```

## Requirements

Bash 4+, `git`, authenticated `gh`, `jq`, and a headless coding-agent CLI.

## Install

```bash
install -m 0755 bin/ralph-gh ~/.local/bin/ralph-gh
ralph-gh labels
```

Run it from a checkout of the target repository.

## Configure

The agent command must accept the prompt as its final argument:

```bash
export RALPH_AGENT_CMD='opencode run'
# alternatives, after checking their current CLI syntax:
# export RALPH_AGENT_CMD='codex exec --full-auto'
# export RALPH_AGENT_CMD='claude --print --dangerously-skip-permissions'
```

Deterministic completion is mandatory:

```bash
export RALPH_VERIFY_CMD='uv run ruff check . && uv run pytest -q'
```

## Run

```bash
# Put ralph-ready on issue #42 first
ralph-gh run 42 --reviewer YOUR_GITHUB_LOGIN

# Or claim the oldest ready issue
ralph-gh next --reviewer YOUR_GITHUB_LOGIN

ralph-gh status 42
```

## Labels

- `ralph-ready`
- `ralph-running`
- `ralph-review`
- `ralph-failed`

## Human steering

Comment on the issue while the loop runs. Before every fresh iteration, `ralph-gh` refreshes the issue body and comments into `.ralph/issue.md`.

## Security

The controller owns `gh`. It removes `GH_TOKEN`, `GITHUB_TOKEN`, and `GITHUB_PAT` from the agent process. For stronger isolation, run each worktree/agent invocation in Docker, Podman, or Apptainer. Protect the default branch and require human approval.

## MVP limitations

- Local checkout required.
- One shell verification command.
- No daemon or atomic multi-worker claim yet.
- No automatic PR-review feedback loop yet.
- Agent CLI conventions differ, so configure `RALPH_AGENT_CMD`.
