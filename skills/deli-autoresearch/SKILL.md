---
name: deli-autoresearch
description: Use when running long-horizon autonomous research, writing, exploration, benchmark, or experiment tasks that must continue unattended and avoid cognitive loops, stalling, or silent loop death.
---

# Deli AutoResearch

## Overview

Deli AutoResearch is a protocol harness for long-horizon autonomous tasks. It turns a loose agent loop into an engineered system with persisted state, enforced direction diversity, quantitative stall detection, and watchdog recovery.

Core principle: **do not trust a long-running agent to remember, self-evaluate, or stay alive. Externalize state, separate workers from evaluators, and restart from curated files.**

## When to Use

Use this for:

- Multi-hour or multi-day research, literature, writing, benchmark, or exploration tasks
- Autonomous runs where the user explicitly authorized zero-interaction operation
- Tasks prone to repeating the same approach, waiting for feedback, or dying silently
- Multi-agent exploration where each worker needs a distinct direction
- Experiment loops that require submit, monitor, diagnose, fix, and resubmit cycles

Do not use this for:

- Small one-shot code edits or simple debugging
- Work that requires frequent human judgment before acting
- External actions not explicitly authorized by the user
- Tasks without any verifiable progress metric or validation step

## Behavioral Contract

When operating under this skill, follow these constraints for the scoped run:

1. **Zero interaction after launch** — do not ask questions, enter plan mode, or stop on “should I continue?” Resolve ambiguity and log the decision.
2. **Ready means execute** — if preparation is complete, proceed with routine actions: submit, monitor, fix, resubmit, verify, and restart monitors.
3. **Persist state to files** — progress, findings, directions, and logs live in `{task}/state/` and `{task}/logs/`, not in chat memory.
4. **Fresh session over resume** — start each worker iteration from curated state files. Avoid accumulated context and `resume` unless the user specifically requires it.
5. **Separate guardian and worker duties** — watchdogs may liveness-check, restart, and nudge only. They must not edit worker findings or impersonate worker summaries.
6. **External side effects need scope** — pushing, posting, emailing, spending, deleting, or modifying third-party resources requires prior explicit authorization.

## Directory Layout

Create one task directory per autonomous job:

```text
{task}/
  state/
    task_spec.md
    progress.json
    findings.jsonl
    directions_tried.json
    iteration_log.jsonl
  logs/
    work.jsonl
    orchestrator.jsonl
    heartbeat.jsonl
```

### `state/task_spec.md`

Contains goal, scope, non-goals, success criteria, allowed actions, validation rules, stop conditions, and escalation conditions.

### `state/progress.json`

```json
{
  "iteration": 0,
  "total_findings": 0,
  "status": "initialized",
  "stale_count": 0,
  "last_seen": "2026-06-30T00:00:00Z",
  "last_metric": null
}
```

### `state/findings.jsonl`

Append-only evidence records:

```json
{"ts":"...","iteration":1,"claim":"...","evidence":"...","source":"...","confidence":"low|medium|high"}
```

### `state/directions_tried.json`

```json
[
  {"iteration":1,"direction":"survey primary sources first","result":"3 new findings","stalled":false}
]
```

### Logs

Use this JSONL format:

```json
{"ts":"...","source":"orchestrator|worker|heartbeat","level":"info|warn|error|decision","event":"...","detail":"..."}
```

Use `level=decision` whenever resolving ambiguity without user input.

## Orchestrator Loop

The orchestrator owns scheduling and stall decisions.

Each callback must:

1. Update its own `last_seen` immediately.
2. Read `task_spec.md`, `progress.json`, and `directions_tried.json`.
3. Decide whether the previous iteration made measurable progress.
4. Update `stale_count`.
5. Choose a direction that differs structurally from all previous directions.
6. Launch worker agent(s) with explicit deliverables and completion criteria.
7. Verify output and update state files.
8. Run validation before the next iteration.

### Orchestrator Prompt Template

```text
You are the orchestrator for {task}. Zero interaction is authorized within the task_spec scope.

Read task_spec.md, progress.json, directions_tried.json, and recent iteration_log.jsonl.
First update orchestrator last_seen in progress/logs.

Decide whether the previous iteration produced new verifiable findings or metric improvement.
If no progress, increment stale_count. If stale_count >= 2, choose a structural pivot, not a tactical tweak.

Launch a fresh worker session with:
- one distinct direction not present in directions_tried.json
- hard cap: 15 rounds or 30 minutes
- deliverable: append verified findings and iteration summary to state files
- completion criteria from task_spec.md

Do not ask the user questions. Log decisions with level=decision.
```

## Worker Pattern

Workers execute, but do not judge global progress.

Worker requirements:

- Read only the task spec and curated state needed for the iteration
- Use the assigned direction; do not invent a similar duplicate direction
- Produce verifiable findings or a clear null result
- Append results to files
- Stop at the round/time cap
- Do not summarize to the user directly during unattended mode

### Worker Prompt Template

```text
You are a fresh worker for {task}, iteration {n}.

Read task_spec.md, assigned direction, and relevant prior directions.
Assigned direction: {direction}

Rules:
- Do not ask questions.
- Do not reuse a prior direction.
- Produce verifiable findings only.
- Append findings to findings.jsonl.
- Append an iteration summary to iteration_log.jsonl.
- Log decisions to work.jsonl with level=decision.
- Cap at 15 rounds or 30 minutes.

Completion criteria: at least one new verifiable finding, or a documented null result explaining why this direction failed; state files updated; validation completed.
```

## Stall Detection and Pivoting

Use quantitative rules instead of self-assessment.

| Condition | Action |
|---|---|
| 0 new findings | `stale_count += 1` |
| metric drop | `stale_count += 1` |
| progress made | reset or decrement `stale_count` |
| `stale_count >= 2` | force structural pivot |
| `stale_count >= 4` | mark structurally stuck and prepare human-facing report |

**Pivot structure, not tactics.**

Bad pivot: “search harder with better keywords.”

Good pivots:

- Change source class: papers → code repos → issue trackers → datasets
- Change decomposition: topic-first → claim-first → counterexample-first
- Change evaluator: self-review → independent verification worker
- Change environment: local assumptions → external validation
- Change objective: broad survey → adversarial refutation

## Direction Diversity

A valid new direction must differ in at least one structural dimension:

- Source type
- Decomposition strategy
- Evidence standard
- Evaluation method
- Search space
- Output representation
- Failure hypothesis

If two directions would produce similar actions, they are not diverse enough.

## Heartbeat Watchdog

The watchdog is a guardian, not a worker.

It may only:

1. Check liveness.
2. Restart missing loops.
3. Nudge stalled workers.

It must not edit findings, rewrite task specs, report worker results as its own, or make external side effects outside the authorized scope.

### Heartbeat Prompt Template

```text
You are the heartbeat watchdog for {task_root}.

For each task:
1. Write heartbeat timestamp to logs/heartbeat.jsonl.
2. Read progress.json last_seen fields only.
3. If a loop is stale beyond interval * 3, restart it with the orchestrator prompt.
4. If progress has not changed for more than 2 hours and last output is a question or waiting state, launch a nudge worker.
5. After 3 consecutive nudges with no progress, stop nudging and mark structurally stuck.

Do not inspect or modify worker findings. Do not ask the user questions.
```

## Subagent Scheduling Patterns

| Pattern | Use | Requirement |
|---|---|---|
| Goal-driven worker | Normal iteration | assigned direction + verifiable findings |
| Parallel exploration | Complex subproblem | launch distinct roles in one tool call |
| Refutation worker | Quality hardening | try to disprove current findings |
| Cross-domain analogy | Escape local optimum | import methods from another domain |
| Verification worker | QA | audit evidence chain independently |
| Experiment worker | Long compute | submit, poll, diagnose, fix, resubmit |

For parallel work, assign explicit directions. Never rely on agents to self-diversify.

## Validation

Between iterations, run the task’s validation step:

- Code: test, build, lint, typecheck, benchmark, or reproducible script
- Research: source verification, citation check, evidence audit, duplicate detection
- Writing: outline consistency, claim-evidence mapping, independent review
- Experiments: metric comparison, artifact existence, log inspection

Verify citation-like or source-like claims every 20 entries. Do not batch large unverified claim sets.

## Escalation

Escalate with a concise report when:

- `stale_count >= 4`
- Required external dependency is unavailable
- Validation cannot run
- Evidence sources conflict and no rule resolves them
- Authorized action scope is insufficient for the next necessary step

Report format:

```text
Status: structurally stuck | blocked | needs authorization
Task: ...
Last progress: ...
Tried directions: ...
Blocking issue: ...
Recommended next structural pivot: ...
Needed human input or authorization: ...
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Ending with “should I continue?” | Continue within authorized scope and log the decision |
| Reusing the same search strategy | Force a structural pivot |
| Letting workers judge their own progress | Orchestrator computes progress from state files |
| Keeping state in chat | Write state files every iteration |
| Using resume for every iteration | Start fresh and inject curated state |
| Watchdog edits findings | Watchdog only checks, restarts, nudges |
| No validation between loops | Add a concrete validation command or evidence check |
| Infinite loop without cap | Use round/time/stale/stop limits |

## Minimal Startup Checklist

1. Confirm unattended scope is authorized.
2. Create `{task}/state/` and `{task}/logs/`.
3. Write `task_spec.md` with goal, allowed actions, validation, and stop conditions.
4. Initialize `progress.json`, `findings.jsonl`, `directions_tried.json`, and `iteration_log.jsonl`.
5. Start orchestrator loop.
6. Start independent heartbeat.
7. Validate after every iteration.
8. Escalate instead of silently abandoning blocked work.
