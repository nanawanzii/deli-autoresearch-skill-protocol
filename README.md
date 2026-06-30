# Deli AutoResearch

A protocol harness for **long-horizon autonomous tasks** (hours to days). It ships no executable code — it prescribes battle-tested conventions for state persistence, stall detection, direction diversity, and watchdog recovery so an agent loop can run unattended without falling into cognitive loops, stalling on "should I continue?", or dying silently.

## Why

Long-running agents fail in three recurring ways:

1. **Cognitive loop** — same direction retried with diminishing returns.
2. **Stalling** — agent finishes a chunk, summarizes, and waits forever.
3. **Runtime fragility** — context compaction or session close kills the loop silently.

The root cause is missing engineering scaffolding, not model capability. This harness supplies that scaffolding.

## What's inside

A single skill — `deli-autoresearch` — defining:

- A zero-interaction behavioral contract
- A state/log file layout (`progress.json`, `findings.jsonl`, `directions_tried.json`, ...)
- Orchestrator / worker / heartbeat prompt templates
- Quantitative stall detection and "pivot structure, not tactics" rules
- Direction-diversity enforcement
- Subagent scheduling patterns, validation, and escalation rules

## Install

Add this repo as a plugin marketplace, then install:

```text
/plugin marketplace add <your-git-url>
/plugin install deli-autoresearch@deli-autoresearch
```

Or copy the skill directly into your skills directory:

```text
skills/deli-autoresearch/SKILL.md
```

## Use

Invoke the skill when starting an unattended long-horizon task:

```text
/skill deli-autoresearch
```

Then initialize a task directory, write `task_spec.md`, start the orchestrator loop and an independent heartbeat watchdog. See the skill body for full templates.

## License

MIT