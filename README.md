# ralph-gh

A minimal GitHub-native Ralph loop. GitHub issues replace PRD JSON files; labels form the queue/state machine; issue comments allow live human steering; fresh agent processes prevent context rot; Git commits and `.ralph/progress.md` retain memory.

## Workflow

The GitHub issue is the specification: there is no separate PRD JSON file. Labels
form a small state machine, while Git commits and `.ralph/progress.md` carry state
between otherwise fresh agent processes.

```text
ralph-ready issue
        |
        v
create isolated worktree on ralph/issue-<number>
        |
        v
run a fresh agent process for one iteration
        |
        v
run deterministic verification
        |
        +-- failure --> refresh context and start a fresh iteration
        |
        +-- success --> push branch --> open draft PR --> request human review
```

When `ralph-gh run` starts, the controller checks that the issue is ready,
changes its label from `ralph-ready` to `ralph-running`, and creates an isolated
worktree from the selected base branch. The working branch is named
`ralph/issue-<number>`.

For each iteration, a new agent process reads:

- the current issue body and all issue comments from `.ralph/issue.md`;
- repository instructions such as `AGENTS.md`;
- durable notes left in `.ralph/progress.md`;
- recent Git history and the current diff.

The agent makes a small coherent change, runs focused checks, commits its work,
and records useful context for the next iteration. The controller then runs
`RALPH_VERIFY_CMD`. This command—not an agent message such as `COMPLETE`—decides
whether the issue is finished. If it fails, the controller refreshes the issue
and comments before launching another fresh agent process, up to the configured
iteration limit.

After verification passes, the controller commits any remaining tracked work,
pushes the branch, and opens a draft PR. It replaces `ralph-running` with
`ralph-review` and requests the configured reviewer. The PR is never merged
automatically. If the run exits unsuccessfully, the issue is labeled
`ralph-failed` so it can be inspected and returned to `ralph-ready` for another
attempt.

## Requirements

Bash 4+, `git`, authenticated `gh`, `jq`, and a headless coding-agent CLI.

## Install

```bash
install -m 0755 bin/ralph-gh ~/.local/bin/ralph-gh
ralph-gh labels
```

Run it from a checkout of the target repository.

## Configure

The agent command must accept the prompt as its final argument. Install and
authenticate the CLI you want to use first, then set `RALPH_AGENT_CMD` as shown
below.

### OpenCode

```bash
export RALPH_AGENT_CMD='opencode run'
```

`opencode run` starts a non-interactive OpenCode session. `ralph-gh` appends its
iteration prompt after `run`.

### Codex

Use [`codex exec`](https://developers.openai.com/codex/cli/reference) for a
non-interactive run and allow writes only inside the worktree:

```bash
export RALPH_AGENT_CMD='codex exec --sandbox workspace-write'
```

Codex reads the appended Ralph prompt as its positional `PROMPT` argument. The
`workspace-write` sandbox lets it edit, test, and commit in the isolated
worktree without giving it unrestricted access to the host. The older
`--full-auto` flag is deprecated and should not be used in new configurations.

Before starting Ralph, confirm that Codex is authenticated and can run in the
target repository:

```bash
codex exec --sandbox workspace-write 'Inspect this repository and exit without making changes.'
```

### Claude Code

Claude Code must use
[`--print`](https://docs.anthropic.com/en/docs/claude-code/cli-usage) so that it
runs non-interactively and exits after the iteration:

```bash
export RALPH_AGENT_CMD='claude --print --dangerously-skip-permissions'
```

`--dangerously-skip-permissions` is needed for a fully unattended loop because
Claude must be able to edit files, run checks, and commit without waiting for
permission prompts. It disables Claude Code's permission checks, so use this
configuration only when the agent process is isolated in a container or VM.
An isolated Git worktree prevents branch conflicts but is not a security
sandbox.

Before starting Ralph, complete Claude Code's normal authentication flow and
test non-interactive mode in the target repository:

```bash
claude --print 'Inspect this repository and exit without making changes.'
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
