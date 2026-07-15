---
name: spark-subagent-driven-development
description: Use only when the user explicitly asks to execute an implementation plan with Spark as the implementer; do not trigger for ordinary subagent-driven development.
---

# Spark Subagent-Driven Development

Route the implementer side of the standard workflow through Spark without
changing the ordinary `subagent-driven-development` model-selection policy.

**REQUIRED BASE SKILL:** Use `superpowers:subagent-driven-development` and
preserve its task briefs, reports, review packages, durable progress ledger,
continuous execution, unified task review, final whole-branch review, and
review loops.

**REQUIRED PLATFORM ADAPTER:** Use `superpowers:using-superpowers` and its
canonical Codex multi-agent adapter. Do not invent or copy Codex dispatch
signatures here.

## Role routing

| Workflow role | Codex role | Model ownership |
|---------------|------------|-----------------|
| Initial implementer for every task | `spark_implementer` | Role config owns model and effort |
| Task and final reviewers | Role required by `superpowers:requesting-code-review` | Role config owns model and effort |
| One allowed escalation | `worker` | Workflow fallback below owns model and effort |

Never route any review through Spark. Never replace a required role with an
untyped agent that inherits the parent model.

## Workflow

For each task:

1. Follow the base skill's task-brief and report-file protocol, then dispatch a
   fresh `spark_implementer` with its `implementer-prompt.md`. Use the adapter's
   canonical Spark task name.
2. On `DONE` or resolved `DONE_WITH_CONCERNS`, generate the review package and
   dispatch a fresh instance of the review role required by
   `superpowers:requesting-code-review`. Use the adapter's canonical task-review
   name and the base `task-reviewer-prompt.md`.
3. Send all Critical and Important findings back to the same active
   implementer, then re-run the unified task review. Do not spawn a replacement
   implementer for review fixes.
4. Complete the task only after the reviewer approves both spec compliance and
   task quality, then update the base workflow's progress ledger.

After every task is complete, run the base workflow's final whole-branch review
with the review role required by `superpowers:requesting-code-review` and its
`code-reviewer.md` template.

Use `a2`, `a3`, and so on only when a fresh reviewer attempt is required. A
follow-up turn on an existing agent keeps its original task name.

## Status and fallback

- `NEEDS_CONTEXT`: provide the missing context to the same Spark implementer.
- Context-related `BLOCKED`: provide materially new context and retry the same
  Spark implementer once.
- A reasoning/architecture `BLOCKED`, or the same blocker after that context
  retry: dispatch exactly one `worker` fallback on `gpt-5.4/high` through the
  adapter, using its canonical fallback task name.
- After fallback takes over, send review fixes back to that same worker.
- If fallback is `BLOCKED`, stop and return the exact blocker to the user.

Do not chain retries, use profiles, downgrade reviewers, or silently fall back
when routing fields are unavailable. Adapter routing errors terminate the
workflow unchanged.

## Prompt templates

- `../subagent-driven-development/implementer-prompt.md`
- `../subagent-driven-development/task-reviewer-prompt.md`
- `../requesting-code-review/code-reviewer.md`
