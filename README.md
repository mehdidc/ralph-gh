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
run verification (if configured)
        |
        +-- failure --> refresh context and start a fresh iteration
        |
        +-- success --> optional automatic review/fix rounds
                              |
                              v
                      push branch --> open draft PR --> request human review
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
and records useful context for the next iteration. When `RALPH_VERIFY_CMD` is
configured, that command—not an agent message such as `COMPLETE`—decides whether
the issue is finished. If it fails, the controller refreshes the issue and
comments before launching another fresh agent process, up to the configured
iteration limit. Without a verification command, a successful agent exit ends
the implementation loop; nonzero exits are retried up to the same limit.

During implementation, the agent maintains `.ralph/pr-summary.md` from the
actual completed diff. After the implementation loop completes, the controller
commits any remaining tracked work.
If automatic review is configured, it runs the bounded review/fix loop before
pushing; otherwise it proceeds immediately. It then pushes the branch, opens a
draft PR, replaces `ralph-running` with `ralph-review`, and requests the
configured reviewer. The PR is never merged automatically. If the run exits
unsuccessfully, the issue is labeled `ralph-failed` so it can be inspected and
returned to `ralph-ready` for another attempt.

The draft PR body contains the agent's final implementation summary, the actual
Git diff stat, the configured verification command, and automatic-review
status. If the agent did not produce a summary, commit subjects from the branch
are used as a deterministic fallback.

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

### Optional automatic review

Automatic review is disabled by default. To review the verified changes before
the branch is pushed, configure a second non-interactive agent command:

```bash
export RALPH_REVIEW_AGENT_CMD='codex exec --sandbox read-only'
export RALPH_MAX_REVIEW_ROUNDS=3
```

The reviewer gets a read-only prompt and either reports actionable findings or
ends with `RALPH_REVIEW_APPROVED`. When it reports findings, a fresh
implementation agent addresses them, configured verification runs again, and
another fresh reviewer checks the result. The branch is pushed only after
approval and a final successful verification when configured. The run fails if
review is not approved within the configured number of rounds.

The equivalent command-line options are:

```bash
ralph-gh run 42 \
  --review-agent-cmd 'codex exec --sandbox read-only' \
  --max-review-rounds 3
```

Use `--no-review` to disable automatic review for one run even when
`RALPH_REVIEW_AGENT_CMD` is set. Human review of the resulting draft PR is
still required whether or not automatic review is enabled.

Deterministic verification is optional but recommended:

```bash
export RALPH_VERIFY_CMD='uv run ruff check . && uv run pytest -q'
```

Leave `RALPH_VERIFY_CMD` unset to use the agent's exit status, or pass
`--no-verify` to disable an inherited verification command for one run.

## Run

```bash
# Put ralph-ready on issue #42 first
ralph-gh run 42 --reviewer YOUR_GITHUB_LOGIN

# Or claim the oldest ready issue
ralph-gh next --reviewer YOUR_GITHUB_LOGIN

ralph-gh status 42
```

## Daemon

Run a sequential worker that continuously processes the oldest
`ralph-ready` issue:

```bash
ralph-gh daemon --reviewer YOUR_GITHUB_LOGIN
```

The daemon runs one issue at a time using the normal `run` workflow. After a
run succeeds, it immediately claims the next ready issue. If an issue fails,
the normal failure handling labels it `ralph-failed`; the daemon waits for the
poll interval and continues watching the queue. When no issue is ready, it
waits 60 seconds by default before checking again.

Configure the interval with an environment variable or command-line option:

```bash
export RALPH_POLL_INTERVAL=30
ralph-gh daemon --poll-interval 30
```

All normal execution options are supported, including `--agent-cmd`,
`--verify-cmd`, `--max-iterations`, optional automatic review settings,
`--no-pr`, and `--keep-worktree`. Stop the daemon with `Ctrl-C` or send it
`SIGTERM`. The daemon is a single sequential worker; claiming issues is not
atomic across multiple daemon processes.

## Labels

- `ralph-ready`
- `ralph-running`
- `ralph-review`
- `ralph-failed`

## Human steering

Comment on the issue while the loop runs. Before every fresh iteration, `ralph-gh` refreshes the issue body and comments into `.ralph/issue.md`.

## Security

The controller owns `gh`. It removes `GH_TOKEN`, `GITHUB_TOKEN`, and `GITHUB_PAT` from the agent process. For stronger isolation, run each worktree/agent invocation in Docker, Podman, or Apptainer. Protect the default branch and require human approval.

When a run fails, the controller applies `ralph-failed` and posts an issue
comment containing the failed stage, a safe reason, the exit status, a bounded
excerpt from the failing subprocess, retry instructions, and whether the
worktree was retained. URL credentials and common GitHub token formats are
redacted. Full commands and logs are not copied into the comment.

## MVP limitations

- Local checkout required.
- At most one optional shell verification command.
- No atomic multi-worker claim yet; run only one daemon per repository.
- No automatic feedback loop for review comments posted after the PR opens.
- Agent CLI conventions differ, so configure `RALPH_AGENT_CMD`.
