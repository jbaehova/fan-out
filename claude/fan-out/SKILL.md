---
name: fan-out
description: Orchestrate Claude Code subagents for explicit fan-out work across discovery, implementation, testing, validation, review, and synthesis. Use only when the user invokes /fan-out or asks Claude to fan out, spawn subagents, delegate work in parallel, run separate agents per question/subsystem/file group/hypothesis, or synthesize parallel work. Do not use for ordinary complex tasks unless parallel delegation is explicitly requested.
---

# FAN-OUT

Use this skill as a parallel delegation protocol. Split the user's request into independent workstreams, delegate bounded tasks to Claude Code subagents, wait for the required results, resolve conflicts, and synthesize one final answer.

## Activation Rules

- Use only when the user explicitly asks for fan-out, subagents, parallel delegation, one-agent-per-item work, or invokes `/fan-out`.
- Do not spawn subagents just because a task is complex.
- Do not fan out for trivial tasks, one-shot shell commands, direct Q&A, small one-file edits, or workflows with strict linear dependencies.
- Use at most 10 subagents. Spawn only as many as genuinely independent workstreams require.
- Use built-in Claude Code subagents by their documented names:
  - `Explore` for read-only codebase discovery, targeted search, architecture mapping, log/test analysis, review, and hypothesis checks.
  - `Plan` only when Claude is already in plan mode and needs read-only research before presenting a plan.
  - `general-purpose` for edit-capable or action-oriented work: bounded implementation, test creation/fixes, validation repairs, and multi-step operations.
- Treat `general-purpose` as the built-in edit-capable subagent. Do not invent alternate built-in agent names.
- For non-trivial coding tasks, actively look for implementation and test workstreams that can be delegated to `general-purpose`. If no edit-capable or validation subagent is used, explain why in the final synthesis.
- For tasks that include both discovery and code changes, treat read-only subagents as the first wave only. After discovery results are integrated, reassess implementation, test, and validation workstreams and spawn `general-purpose` subagents for independent edit-capable scopes whenever practical.
- Do not use custom agents unless the user explicitly names one.
- Do not use Claude Code helper agents such as `statusline-setup` or `claude-code-guide`; they are automatic helpers, not fan-out subagents.

## Claude Subagent Constraints

- Non-fork subagents start with a fresh isolated context. They do not see the full conversation, previously invoked skills, or files already read by the main conversation.
- `Explore` and `Plan` skip `CLAUDE.md` files and the parent session's git status. Restate any required project rules, ignore paths, branch constraints, or safety constraints in their delegation prompt.
- Other built-in and custom subagents load normal `CLAUDE.md` and memory context, but still need a self-contained task prompt.
- `Explore` is read-only and cannot write or edit files. Use it when the value is keeping noisy search output, logs, or broad inspection out of the main context.
- `general-purpose` can use all inherited tools and may edit files. Give every edit-capable task an explicit owned write scope.
- Prefer foreground subagents for short blocking work. Ask Claude to run subagents in the background when independent long-running work can proceed concurrently.

## Lifecycle Protocol

Use fan-out across the whole workflow, not only during discovery.

1. Discover: use `Explore` for independent read-only questions, architecture mapping, risk checks, or log/test analysis.
2. Plan ownership: identify implementation, testing, and validation workstreams before spawning edit-capable agents, then repeat this ownership pass after `Explore` results narrow the fix. Record the intended agent type and owned write scope for each `general-purpose` candidate.
3. Implement: before making edits in the main conversation, run the discovery-to-implementation checkpoint below. Use `general-purpose` for independent code or test changes with disjoint write scopes. Keep implementation in the main conversation when the change is too small, strictly linear, or would require overlapping edits.
4. Validate and review: delegate test authoring, test fixes, targeted reruns, read-only review, or log analysis when those tasks can run independently. Use `Explore` for read-only validation and `general-purpose` for validation tasks that may edit files.
5. Integrate: inspect subagent outputs, resolve conflicts, run or confirm final validation, and synthesize one final answer.

## Discovery-to-Implementation Checkpoint

For integrated tasks that start with investigation and continue into code changes, run this checkpoint after `Explore` results are integrated and before editing:

- Convert findings into concrete implementation candidates: affected files/modules, behavior changes, tests, and validation commands.
- Split candidates into disjoint write scopes and spawn `general-purpose` subagents for any edit-capable scope that can be owned independently, including test-only scopes.
- Spawn independent validation or review workstreams in parallel when they do not depend on unfinished edits.
- Keep implementation in the main conversation only for overlapping, tiny, or strictly sequential changes, and record that reason for final Delegation Coverage.
- Do not count an early read-only wave as sufficient fan-out for a non-trivial coding task when implementation or test edit-capable scopes become available after discovery.

## Plan-Mode Handoff

When `/fan-out` is active while Claude Code is in plan mode, the final plan shown before an implementation confirmation must carry the fan-out requirement into the next turn. A simple user confirmation such as "yes" or "implement this plan" may not restate `/fan-out`, so the plan itself must make fan-out part of the implementation contract.

In the plan, include a concise fan-out implementation note that:

- States that implementation should continue as a fan-out workflow after approval.
- Lists the planned post-approval implementation, test, and validation workstreams with the intended `Explore` or `general-purpose` agent type.
- Defines the owned write scope for every planned edit-capable `general-purpose` subagent.
- Says which work, if any, will stay in the main conversation and why.
- Requires final integration, validation, conflict resolution, and delegation coverage reporting.

Put this note inside the final plan itself, not only in surrounding explanation. Before asking for approval, verify that the plan contains an explicit fan-out implementation note; if it does not, revise the plan before presenting it.

Do not present a fan-out plan that implies the main conversation should perform all implementation after approval unless the work is too small, strictly linear, or has overlapping write scopes. If implementation must stay in the main conversation, say that explicitly in the plan and still delegate independent discovery, test, review, or validation work where practical.

## Workstream Design

Before spawning agents, define independent workstreams.

For each workstream, specify:

- Objective: the concrete question or deliverable.
- Scope: files, modules, subsystems, logs, tests, hypotheses, or validation commands.
- Non-goals: what the agent must not spend time on.
- Agent type: `Explore`, `Plan`, or `general-purpose`.
- Owned write scope: required for every `general-purpose` task that may edit files, including test tasks.
- Output format: the exact structure needed for synthesis.
- Stop condition: when the agent should stop instead of expanding the task.

Split by one of these axes:

- Subsystem or module.
- File group or ownership boundary.
- Risk category, such as correctness, security, tests, performance, or maintainability.
- Test responsibility, such as adding missing coverage, fixing a failing suite, or validating changed behavior.
- Competing hypothesis.
- Independent user-requested checklist item.

Avoid duplicate assignments. If two agents would need the same write scope, do not run them as parallel edit-capable subagents. Stage the work, use `Explore` first, or keep the overlapping implementation in the main thread. Parallel edit-capable agents must own disjoint files, modules, or narrowly defined responsibilities.

## Spawning Rules

- Spawn independent agents in parallel when possible.
- Make each delegation prompt self-contained. Include only the task-local context the subagent needs.
- Name the intended built-in agent type explicitly: `Explore`, `Plan`, or `general-purpose`.
- For `Explore`, specify thoroughness when useful: quick, medium, or very thorough.
- Do not set model, effort, isolation, permissions, or background behavior unless the user requested it or the task has a clear reason.
- While subagents run, do useful non-overlapping work in the main conversation.
- Wait for all required results before final synthesis.
- Close or stop completed background agents after their results are integrated when Claude Code exposes that control.

## Explore Prompt Template

Use `Explore` for read-only work.

```text
Use the Explore subagent.

Task:
[bounded question]

Scope:
[files/modules/tests/logs/hypothesis]

Thoroughness:
[quick | medium | very thorough]

Context the subagent must know:
[project rules, ignore paths, branch constraints, or safety constraints not guaranteed to be available to Explore]

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

## General-Purpose Prompt Template

Use `general-purpose` for bounded implementation, test creation/fixes, validation repairs, or other edit-capable work.

```text
Use the general-purpose subagent.

Task:
[implementation, test, or validation objective]

Owned write scope:
[files/modules the subagent may edit]

Read scope:
[files/modules/tests/logs the subagent may inspect]

Context the subagent must know:
[relevant user request details, project constraints, active branch/worktree notes, and any concurrent workstreams]

Rules:
- You are not alone in the codebase. Other agents or the main conversation may edit nearby files.
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

## Plan Prompt Template

Use `Plan` only while Claude is already in plan mode and the delegated work is read-only research for a plan.

```text
Use the Plan subagent.

Planning question:
[bounded research needed before presenting the plan]

Scope:
[files/modules/tests/logs/hypothesis]

Context the subagent must know:
[project rules or constraints not guaranteed to be available to Plan]

Rules:
- Work read-only.
- Gather only the evidence needed to make the plan concrete.
- Do not propose broad refactors unless the evidence shows they are necessary.

Return:
1. Findings
2. Relevant file references
3. Planning risks or open questions
```

## Conflict Handling

Treat subagent outputs as evidence, not authority.

When agents disagree:

- Prefer direct code evidence, tests, logs, and precise file references.
- Inspect the relevant source yourself if the conflict affects the final answer or implementation.
- Report material conflicts explicitly.
- Do not average conflicting recommendations.
- If edit-capable changes overlap despite the plan, integrate one ownership area at a time and preserve unrelated user or agent changes.

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
[State which Explore, Plan, general-purpose, test, or validation workstreams were used, including whether the post-discovery implementation checkpoint spawned edit-capable subagents. For non-trivial coding tasks with no edit-capable or validation subagent, explain why.]

**Recommended Next Step**
[The most practical next action, or up to three when useful]
```

Do not dump raw subagent logs. Distill results into decisions, evidence, changed files, and next steps.
