---
name: fan-out
description: Use when the user explicitly invokes $fan-out or asks Codex to fan out, spawn subagents, delegate work in parallel, use explorer/worker agents, or assign one agent per question, subsystem, file group, or hypothesis. Orchestrate built-in Codex subagents across discovery, implementation, testing, validation, review, and integration for non-trivial coding, investigation, or delivery tasks, then synthesize one final response. Do not use for ordinary complex tasks unless parallel delegation is explicit.
---

# FAN-OUT

Use this skill as a parallel delegation protocol, not as a persona system. Create task-specific, self-contained prompts for built-in Codex subagents, wait for their results, resolve conflicts, and synthesize one final answer.

## Activation Rules

- Use only when the user explicitly asks for fan-out, subagents, parallel delegation, one-agent-per-item work, or invokes `$fan-out`.
- Do not spawn subagents just because a task is complex.
- Do not fan out for trivial tasks, one-shot shell commands, direct Q&A, small one-file edits, or workflows with a strict linear dependency.
- Use at most 10 subagents. Do not enforce a minimum; spawn only as many as genuinely independent workstreams require.
- Use built-in agents:
  - `explorer` for read-only investigation, review, test/log analysis, architecture mapping, and hypothesis checking.
  - `worker` for bounded implementation, test creation/fixes, validation repairs, or other file edits when the owned write scope can be isolated.
  - `default` only when neither `explorer` nor `worker` fits the delegated task.
- For non-trivial coding tasks, actively look for implementation and test workstreams that can be delegated to workers. If no worker or validation subagent is used, explain why in the final synthesis.
- For tasks that include both discovery and code changes, treat discovery agents as the first wave only. After discovery results are integrated, reassess implementation, test, and validation workstreams and spawn workers for independent edit-capable scopes whenever practical.
- Do not create unnecessary personas.
- Do not use custom agents unless the user explicitly names one.

## Lifecycle Protocol

Use fan-out across the whole workflow, not only during discovery.

1. Discover: use explorers for independent read-only questions, architecture mapping, risk checks, or log/test analysis.
2. Plan ownership: identify possible implementation, testing, and validation workstreams before spawning agents, then repeat this ownership pass after explorer results narrow the fix. Record the intended agent type and owned write scope for each worker candidate.
3. Implement: before making edits in the main thread, run the discovery-to-implementation checkpoint below. Default to worker subagents for independent code or test changes with disjoint write scopes. Keep implementation in the main thread only when the change is too small, strictly linear, or would require overlapping edits.
4. Validate and review: delegate test authoring, test fixes, targeted reruns, read-only review, or log analysis when those tasks can run independently. Use explorers for read-only validation and workers for validation tasks that may edit files.
5. Integrate: review worker outputs, resolve conflicts, run or confirm final validation, and synthesize one final answer.

## Discovery-to-Implementation Checkpoint

For integrated tasks that start with investigation and continue into code changes, run this checkpoint after explorer results are integrated and before editing:

- Convert findings into concrete implementation candidates: affected files/modules, behavior changes, tests, and validation commands.
- Split candidates into disjoint write scopes and spawn `worker` subagents for any scope that can be owned independently, including test-only scopes.
- Spawn independent validation or review workstreams in parallel when they do not depend on unfinished edits.
- Keep implementation in the main thread only for overlapping, tiny, or strictly sequential changes, and record that reason for final Delegation Coverage.
- Do not count an early explorer-only wave as sufficient fan-out for a non-trivial coding task when implementation or test worker scopes become available after discovery.

## Plan-Mode Handoff

When `$fan-out` is active while Codex is in plan mode, the final plan shown before an implementation confirmation must carry the fan-out requirement into the next turn. A simple user confirmation such as "yes" or "implement this plan" may not restate `$fan-out`, so the plan itself must make fan-out part of the implementation contract.

In the plan, include a concise fan-out implementation note that:

- States that implementation should continue as a fan-out workflow after approval.
- Lists the planned implementation, test, and validation workstreams with the intended `explorer` or `worker` agent type.
- Defines the owned write scope for every planned `worker`.
- Says which work, if any, will stay in the main thread and why.
- Requires final integration, validation, conflict resolution, and delegation coverage reporting.

Put this note inside the final plan itself, not only in surrounding explanation. Before asking for approval, verify that the plan contains an explicit fan-out implementation note; if it does not, revise the plan before presenting it.

Do not present a fan-out plan that implies the main thread should perform all implementation after approval unless the work is too small, strictly linear, or has overlapping write scopes. If implementation must stay in the main thread, say that explicitly in the plan and still delegate independent discovery, test, review, or validation work where practical.

## Workstream Design

Before spawning agents, define independent workstreams.

For each workstream, specify:

- Objective: the concrete question or deliverable.
- Scope: files, modules, subsystems, logs, tests, hypotheses, or validation commands.
- Non-goals: what the agent must not spend time on.
- Agent type: `explorer` or `worker`.
- Owned write scope: required for every worker, including test workers.
- Output format: the exact structure needed for synthesis.
- Stop condition: when the agent should stop instead of expanding the task.

Split by one of these axes:

- Subsystem or module.
- File group or ownership boundary.
- Risk category, such as correctness, security, tests, performance, or maintainability.
- Test responsibility, such as adding missing coverage, fixing a failing suite, or validating a changed behavior.
- Competing hypothesis.
- Independent user-requested checklist item.

Avoid duplicate assignments. If two agents would need the same write scope, do not run them as parallel workers. Stage the work, use explorers first, or keep the overlapping implementation in the main thread. Parallel workers must own disjoint files, modules, or narrowly defined responsibilities.

## Spawning Rules

- Spawn independent agents in parallel when possible.
- Make each subagent prompt self-contained. Omit conversation history unless the agent truly needs it.
- Do not set model, reasoning effort, or service tier unless the user requested it or there is a clear task-specific reason.
- While agents run, do useful non-overlapping work in the main thread.
- Wait for all required results before final synthesis.
- Close completed agents after their results are integrated.

## Explorer Prompt Template

Use `explorer` for read-only work.

```text
You are an explorer subagent in a Codex fan-out workflow.

Task:
[bounded question]

Scope:
[files/modules/tests/logs/hypothesis]

Rules:
- Work read-only. Do not edit files.
- Stay inside the assigned scope.
- Use concrete evidence.
- Prefer file paths and line numbers for code findings.
- Do not solve adjacent problems unless they directly affect the assigned question.
- If the evidence is inconclusive, say exactly what is unknown.

Return a concise result with:
1. Answer
2. Evidence
3. Risks or unknowns
4. Recommended next probe, if any
```

## Worker Prompt Template

Use `worker` for bounded implementation, test creation/fixes, validation repairs, or file edits with a clear owned write scope.

```text
You are a worker subagent in a Codex fan-out workflow.

Task:
[implementation, test, or validation objective]

Owned write scope:
[files/modules the worker may edit]

Read scope:
[files/modules/tests/logs the worker may inspect]

Context:
[relevant discovery findings, user request details, project constraints, active branch/worktree notes, and concurrent workstreams]

Rules:
- You are not alone in the codebase. Other agents or the main agent may edit nearby files.
- Do not revert, overwrite, or broad-format changes made by others.
- Edit only the owned write scope unless blocked.
- If the assigned scope is insufficient, stop and report the blocker instead of expanding edits.
- Follow existing codebase patterns.
- Keep changes narrow and behavior-focused.
- Run the narrowest relevant validation available, especially for test-focused assignments.

Return a concise result with:
1. Changed files
2. Summary of changes
3. Validation run and result
4. Risks, blockers, or follow-up needed
```

## Conflict Handling

Treat subagent outputs as evidence, not authority.

When agents disagree:

- Prefer direct code evidence, tests, logs, and precise file references.
- Inspect the relevant source yourself if the conflict affects the final answer or implementation.
- Report material conflicts explicitly.
- Do not average conflicting recommendations.
- If worker changes overlap despite the plan, integrate one ownership area at a time and preserve unrelated user or agent changes.

## Final Synthesis

Use this structure when relevant, adapting labels to the user's language:

```text
**Result**
[Key findings or implementation result]

**Agent Conflicts**
[Say none if there were no material conflicts; otherwise summarize the conflict and decision]

**Changed Files**
[List changed files and summaries if files were modified; otherwise say none]

**Validation**
[Tests or commands run and results; if not run, explain why]

**Delegation Coverage**
[State which explorer, worker, test, or validation workstreams were used, including whether the post-discovery implementation checkpoint spawned workers. For non-trivial coding tasks with no worker or validation subagent, explain why.]

**Recommended Next Step**
[The most practical next action, or up to three when useful]
```

Do not dump raw subagent logs. Distill results into decisions, evidence, changed files, and next steps.
