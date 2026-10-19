---
name: adaptive-orchestrator
description: Coordinate cost-conscious work across models, with optional isolated subagents when parallelism or separation justifies their overhead.
---

# Adaptive Orchestrator

Use this skill when the user explicitly invokes `$adaptive-orchestrator`.
Optimize total effort and cost, including dispatch, context transfer, integration,
and review. Keep architectural decisions and difficult reasoning with the primary
model; prefer the worker model for bounded tasks with clear acceptance criteria.

## Model preferences

- **Worker:** OpenAI `gpt-6-luna`, effort `max`. Prefer it for exploration,
  routine implementation, integration, and grouped fixes.
- **Primary:** OpenAI `gpt-6.1-sol`, effort `high` by default. Use it for
  orchestration, architecture, difficult blockers, and critical final review.
  Use `medium` or `low` when the task warrants less reasoning.

These are routing preferences, not a claim about current prices or availability.
Discover the live provider/model catalog and supported effort settings before
dispatch. Select the configured provider instance that exposes the requested model.
If a preference is unavailable, state the limitation and use an available suitable
option within the user's constraints. A routing preference does not switch the
active conversation's model.

## Choose the execution mode

- **Local execution:** Complete small tasks in the current session when handing
  them off would cost more than it saves.
- **Direct model delegation:** Hand a bounded task to the worker model without
  creating a separate checkout when the host supports this mode. Use it for quick
  exploration, focused edits, and straightforward integration.
- **Isolated subagent:** Use an autonomous background worker with its own
  worktree/branch for substantial independent tasks, overlapping file ownership,
  or work that benefits materially from parallel execution.

Direct delegation is a host capability. Some hosts implement every delegated call
as a child task; use their actual semantics rather than assuming a separate
lightweight model-call API exists. When delegation is unavailable, continue locally
and report the limitation if it affects delivery.

When using T3 Code, discover models through `orchestrator_capabilities`. Prefer
native delegation for same-provider work only when it supports the selected model
and required settings. Use `delegate_task` for unsupported native selections,
cross-provider work, explicitly T3-owned child tasks, and any task requiring
`isolation: "worktree"`. Keep delegated work as child tasks; ordinary top-level
threads require a separate user request. Retain task IDs for status and cancellation.

## Workflow

1. **Map dependencies.** Turn the requested outcome into a task graph. For each
   task, identify prerequisites, expected output, acceptance criteria, and likely
   file ownership. The **frontier** contains tasks whose prerequisites are met.
   Keep this map in the current plan; use a shared note only when handoffs need it.

2. **Resolve uncertainty.** Use the worker model for focused exploration when it
   reduces uncertainty cheaply. Use an exploration subagent only for a broad,
   separable search. Record findings as file paths, symbols, and concise decisions
   so later workers can start from those pointers.

3. **Set the integration baseline.** Let the primary model settle shared contracts
   and the minimum structure other tasks depend on. Reuse the current checkout for
   simple work; establish an integration branch when separate worktrees need a
   common target. Record the baseline commit for isolated tasks. Worktree tasks
   start from committed state; uncommitted parent edits need an explicit handoff
   or must be integrated before dispatch.

4. **Dispatch ready tasks.** Choose the least costly suitable mode for each task
   on the frontier. Parallelize only work that can proceed independently; give
   each shared-checkout writer exclusive file ownership. Assign isolated workers
   a baseline and ask them to check for integration changes before starting and
   before reporting completion. If the baseline changes, reconcile against it
   before accepting their result.

5. **Integrate and advance.** Check each result against its acceptance criteria,
   integrate accepted changes, and run the relevant checks. Use the worker model
   for straightforward merges; use a merger subagent only when separate workspaces
   make it useful. Mark dependencies satisfied only after integration and
   verification, then dispatch newly ready work. Keep dispatches bounded by
   available concurrency and stop duplicate or obsolete tasks.

6. **Escalate a specific blocker.** When a worker cannot resolve a task, request
   its evidence, attempted approaches, and unresolved question. Let the primary
   model solve that question or revise the shared contract, then return the
   remaining routine work to the worker. Escalate repeated failures instead of
   retrying the same brief indefinitely.

7. **Review the integrated result.** Have the primary model assess the complete
   outcome, cross-task contracts, and validation evidence. Group compatible fixes
   into one worker delegation, or one repair subagent if isolation is still useful.
   For T3 delegated review rounds, create a fresh `delegate_task` call with the
   original brief, prior findings, responses, and unresolved objections. Give each
   round a distinct `clientRequestId`, reuse it only for retries of that round,
   and track its returned task ID. Recheck the affected behavior after fixes.

8. **Deliver and clean up.** Finish when every requested outcome is integrated
   and verified, or clearly identify what remains blocked and why. Report changes,
   checks, and material limitations. Remove only temporary worktrees and notes
   created for this task after preserving their useful results; retain user work
   and any evidence needed to understand unresolved issues.

## Handoff contract

Keep each handoff short and self-contained enough to execute. Include:

- The outcome and acceptance criteria.
- Context pointers: repository-relative paths, relevant symbols, decisions,
  research notes, and prior artifacts. Include the baseline commit for worktrees.
- Owned files or directories, shared contracts, and integration constraints.
- Required validation and the expected return: changed files, commit or patch,
  check results, and blockers.

Reference existing material instead of copying it. Ensure the recipient can access
each pointer; parent-local notes and uncommitted files may be absent from a child
worktree. Transfer only the missing context needed for the task. Request concise
results at completion or when blocked, and send intermediate messages only when
they change another task's work.
