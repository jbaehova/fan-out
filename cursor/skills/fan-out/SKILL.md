---
name: fan-out
description: Orchestrate Cursor subagents for explicit fan-out work. Use only when the user invokes /fan-out or asks Cursor to fan out, spawn subagents, delegate work in parallel, run separate agents per question/subsystem/file group/hypothesis, or synthesize parallel investigations. Do not use for ordinary complex tasks unless parallel delegation is explicitly requested.
---

# FAN-OUT

Use this skill as a parallel delegation protocol. Split the user's request into independent workstreams, delegate bounded tasks to Cursor subagents, wait for required results, resolve conflicts, and synthesize one final answer.

## Activation Rules

- Use only when the user explicitly asks for fan-out, subagents, parallel delegation, one-agent-per-item work, or invokes `/fan-out`.
- Do not spawn subagents just because a task is complex.
- Do not fan out for trivial tasks, one-shot shell commands, direct Q&A, small one-file edits, or workflows with strict linear dependencies.
- Use at most 10 subagents. Spawn only as many as genuinely independent workstreams require.
- Use Cursor built-in subagents by their documented names:
  - `explore` for codebase search, architecture mapping, read-only investigation, review, and hypothesis checks.
  - `bash` for verbose command execution, test runs, build logs, shell-based validation, and log analysis.
  - `browser` for browser automation, UI inspection, DOM/screenshot-heavy validation, and web workflows through MCP browser tools.
- Cursor has no built-in general edit-capable worker subagent. For parallel implementation or test edits, use the project custom subagent `fan-out-worker` from `.cursor/agents/fan-out-worker.md`.
- If `fan-out-worker` is unavailable, do edit-capable implementation in the main Agent and use built-in subagents only for read-only research, command output, browser checks, or validation.
- Do not create one-off generic personas. Add new custom subagents only when the user explicitly asks or a durable repeated workflow is clear.

## Cursor Subagent Constraints

- Each subagent has its own context and token usage. Use subagents to isolate noisy work, not to avoid thinking in the main Agent.
- Built-in `explore`, `bash`, and `browser` are specialized for context-heavy operations. They are not substitutes for a general coding worker.
- `bash` can run commands, but should not become an implementation path through ad hoc shell edits. Use it for command output isolation and validation unless the user explicitly requests command-driven changes.
- `browser` should return findings, screenshots/DOM evidence summaries, and reproduction steps rather than dumping raw browser snapshots.
- Background subagents may write output under `~/.cursor/subagents/`. Read those outputs when needed to check progress or integrate results.
- Nested subagents can exist in newer Cursor versions, but keep this fan-out protocol top-level unless a delegated task naturally needs its own sub-split.

## Lifecycle Protocol

Use fan-out across the whole workflow, not only during discovery.

1. Discover: use `explore` for independent read-only questions, architecture mapping, risk checks, or codebase search.
2. Plan ownership: identify implementation, testing, browser, shell, and validation workstreams before spawning agents. Record the intended agent type and owned write scope for every `fan-out-worker` candidate.
3. Implement: use `fan-out-worker` for independent code or test changes with disjoint write scopes. Keep implementation in the main Agent when the change is too small, strictly linear, or would require overlapping edits.
4. Validate and review: use `bash` for noisy test/build/log runs, `browser` for UI/browser checks, `explore` for read-only review, and `fan-out-worker` only when validation may require scoped file edits.
5. Integrate: inspect subagent outputs, resolve conflicts, run or confirm final validation, and synthesize one final answer.

## Workstream Design

Before spawning agents, define independent workstreams.

For each workstream, specify:

- Objective: the concrete question or deliverable.
- Scope: files, modules, subsystems, logs, tests, commands, browser flows, hypotheses, or validation targets.
- Non-goals: what the agent must not spend time on.
- Agent type: `explore`, `bash`, `browser`, or `fan-out-worker`.
- Owned write scope: required for every `fan-out-worker` task, including test tasks.
- Output format: the exact structure needed for synthesis.
- Stop condition: when the agent should stop instead of expanding the task.

Split by one of these axes:

- Subsystem or module.
- File group or ownership boundary.
- Risk category, such as correctness, security, tests, performance, UX, or maintainability.
- Test responsibility, such as adding missing coverage, fixing a failing suite, or validating changed behavior.
- Browser flow or UI state.
- Competing hypothesis.
- Independent user-requested checklist item.

Avoid duplicate assignments. If two agents would need the same write scope, do not run them as parallel edit-capable subagents. Stage the work, use `explore` first, or keep the overlapping implementation in the main Agent. Parallel edit-capable agents must own disjoint files, modules, or narrowly defined responsibilities.

## Spawning Rules

- Spawn independent agents in parallel when possible.
- Make each delegation prompt self-contained. Include only the task-local context the subagent needs.
- Name the intended agent type explicitly: `explore`, `bash`, `browser`, or `fan-out-worker`.
- Do not set model, background behavior, or other subagent config unless the user requested it or the task has a clear reason.
- While subagents run, do useful non-overlapping work in the main Agent.
- Wait for all required results before final synthesis.
- If a background subagent has not returned a result, check `~/.cursor/subagents/` or ask Cursor to resume/check the subagent before synthesizing.

## Explore Prompt Template

Use `explore` for read-only codebase work.

```text
Use the explore subagent.

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

## Bash Prompt Template

Use `bash` for noisy command execution or shell-based validation.

```text
Use the bash subagent.

Task:
[command, test suite, build, log inspection, or validation objective]

Commands or command family:
[exact commands if known, or the narrow command discovery scope]

Rules:
- Prefer the narrowest command that answers the question.
- Do not modify files unless the task explicitly allows command-driven changes.
- Capture failing command names, exit codes, and the smallest useful error excerpts.
- If command output is too large, summarize the failure shape and point to the relevant log path.

Return:
1. Commands run
2. Result
3. Key output or failure evidence
4. Recommended next command or fix target, if any
```

## Browser Prompt Template

Use `browser` for UI or web automation.

```text
Use the browser subagent.

Task:
[browser flow, UI validation, web check, or reproduction]

Target:
[URL, route, local server, viewport, credentials state, or browser setup]

Rules:
- Keep browser output summarized. Do not dump raw DOM or screenshot text unless it is the evidence.
- Record exact route, viewport, interaction steps, and observed behavior.
- Distinguish visual evidence from inferred cause.
- Do not edit code.

Return:
1. Flow tested
2. Observed result
3. Evidence
4. Reproduction steps or suspected fix area
```

## Fan-Out Worker Prompt Template

Use `fan-out-worker` for bounded implementation, test creation/fixes, validation repairs, or other edit-capable work. This custom subagent must exist at `.cursor/agents/fan-out-worker.md`.

```text
Use the fan-out-worker subagent.

Task:
[implementation, test, or validation repair objective]

Owned write scope:
[files/modules the subagent may edit]

Read scope:
[files/modules/tests/logs the subagent may inspect]

Context the subagent must know:
[relevant user request details, project constraints, active branch/worktree notes, and concurrent workstreams]

Rules:
- You are not alone in the codebase. Other agents or the main Agent may edit nearby files.
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

## Fallback When No Worker Exists

If `fan-out-worker` is not available:

- Do not pretend `explore`, `bash`, or `browser` can perform general implementation work.
- Keep edit-capable implementation in the main Agent.
- Still fan out read-only discovery, command validation, browser checks, and log analysis.
- In the final synthesis, explicitly say implementation stayed in the main Agent because Cursor has no built-in general edit-capable subagent and `fan-out-worker` was unavailable.

## Conflict Handling

Treat subagent outputs as evidence, not authority.

When agents disagree:

- Prefer direct code evidence, tests, logs, screenshots, browser observations, and precise file references.
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
[Tests, browser checks, builds, or commands run and results; if not run, explain why]

**Delegation Coverage**
[State which explore, bash, browser, fan-out-worker, test, or validation workstreams were used. For non-trivial coding tasks with no edit-capable or validation subagent, explain why.]

**Recommended Next Step**
[The most practical next action, or up to three when useful]
```

Do not dump raw subagent logs. Distill results into decisions, evidence, changed files, and next steps.
