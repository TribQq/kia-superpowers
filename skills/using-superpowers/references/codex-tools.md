# Codex Tool Mapping

Skills use Claude Code tool names. When you encounter these in a skill, use the
Codex equivalent below.

## Contents

- [Basic mapping](#basic-mapping)
- [Canonical multi-agent adapter](#canonical-multi-agent-adapter)
- [Environment detection](#environment-detection)
- [Codex App finishing](#codex-app-finishing)

## Basic mapping

| Skill references | Codex equivalent |
|-----------------|------------------|
| `Task` tool | Canonical multi-agent adapter below |
| Multiple `Task` calls | Multiple adapter dispatches when parallel work is allowed |
| `TodoWrite` | `update_plan` |
| `Skill` tool | Skills load natively; follow their instructions |
| `Read`, `Write`, `Edit` | Native file tools |
| `Bash` | Native shell tools |

## Canonical multi-agent adapter

This section is the single owner of Codex subagent tool signatures for
Superpowers skills. Workflow skills own roles, prompts, ordering, and retry
policy; they must not copy these signatures.

Enable multi-agent support in `~/.codex/config.toml`:

```toml
[features]
multi_agent = true
```

### Invariants

1. Select the branch from the available tool namespace and parameter schema,
   never from a Codex version number.
2. Inspect the dispatch schema before spawning. Every role-pinned dispatch
   requires `agent_type`. An explicit fallback model also requires `model` and
   `reasoning_effort`.
3. Never drop a required routing field, retry as an untyped agent, or accept
   implicit inheritance of the parent model.
4. Agent TOML files own role-pinned model and effort values. Omit `model` and
   `reasoning_effort` when `agent_type` selects such a role. Pass them only for
   an explicit workflow fallback that has no pinned custom role.
5. Give fresh agents only task-local context. Continue an existing agent for
   questions, review fixes, or context retries.
6. Close implementer and reviewer agents when their work is finished whenever
   the selected lifecycle API exposes a close operation.

Use these exact errors and stop the workflow:

```text
Codex multi-agent routing error: multi_agent_v1.spawn_agent could not be loaded through tool search.
Codex multi-agent routing error: <namespace>.spawn_agent does not expose required field(s): <fields>. Refusing to spawn an untyped agent that would inherit the parent model.
Codex multi-agent routing error: <namespace>.spawn_agent rejected required routing field(s): <fields>. Refusing silent fallback.
```

### Task names

Use lowercase task names with a task number and per-role attempt number:

| Role | Name |
|------|------|
| Spark implementer | `spark_impl_t<N>_a<N>` |
| Task reviewer | `task_review_t<N>_a<N>` |
| Explicit fallback | `fallback_t<N>_a<N>` |

The collaboration branch sends this value as `task_name`. The
`multi_agent_v1` branch, whose spawn schema has no `task_name`, starts the
message with `Task name: <name>` so traces retain the same logical identity.

### `multi_agent_v1` branch

Choose this branch when the available tools expose the `multi_agent_v1`
namespace and its spawn schema uses `message` plus optional `agent_type`.

The v1 tools may be deferred. If spawn is not directly visible, use the
available tool-search surface to find and load `multi_agent_v1.spawn_agent`.
In Code Mode, inspect `ALL_TOOLS` to locate the tool-search method; absence from
the initially listed direct tools is not evidence that spawn is unavailable.
If tool search is absent or does not return the namespace, emit the first exact
error above and stop.

Dispatch and lifecycle signatures:

```text
multi_agent_v1.spawn_agent(agent_type="<role>", message="Task name: <name>\n<task-local prompt>")
multi_agent_v1.send_input(target="<agent_id>", message="<follow-up>")
multi_agent_v1.wait_agent(targets=["<agent_id>"])
multi_agent_v1.close_agent(target="<agent_id>")
```

- Use the returned `agent_id` for targeted follow-up, wait, and close calls.
- Wait for the specific critical-path agent; do not substitute code-exec
  `wait`, which resumes yielded exec cells rather than subagents.
- Close a completed v1 agent after its task and review-fix loop no longer need
  its context.

### `collaboration` branch

Choose this branch when the available dispatch schema requires `task_name` and
`message`, supports `fork_turns`, and continuation uses `followup_task`.

Before dispatch, verify that the visible schema includes every routing field
required by the workflow. A role dispatch requires `agent_type`; an explicit
fallback also requires `model` and `reasoning_effort`. If any field is hidden,
missing, or rejected, emit the matching exact error above and stop.

Dispatch and lifecycle signatures:

```text
collaboration.spawn_agent(
  task_name="<name>",
  fork_turns="none",
  agent_type="<role>",
  message="<task-local prompt>"
)

collaboration.spawn_agent(
  task_name="<fallback name>",
  fork_turns="none",
  agent_type="worker",
  model="<explicit model>",
  reasoning_effort="<explicit effort>",
  message="<task-local prompt>"
)

collaboration.followup_task(target="<task_name>", message="<follow-up>")
collaboration.wait_agent()
```

- Always use `fork_turns="none"`; construct the complete task-local prompt
  instead of leaking controller history.
- `wait_agent` is mailbox-based in this branch. After it reports activity,
  consume the delivered mailbox/final-status message rather than expecting
  completed content in the wait result.
- Reuse the canonical task name returned by spawn for every follow-up.

## Review dispatch boundary

Review workflows own the required role, prompt, ordering, and retry policy.
This adapter only enforces the role they select and owns the runtime-specific
dispatch signature. If the required role cannot be enforced, stop with the
routing error instead of substituting an untyped agent.

## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
