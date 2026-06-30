# Deli AutoResearch

[English](./README.md) | [简体中文](./README.zh-CN.md)

A set of conventions for running an agent loop unattended for hours or days. There's no code here — just a skill that tells the agent how to keep its state in files, try a different approach when it gets stuck, and recover when the loop dies.

## The problem

Leave an agent running on its own long enough and it tends to fail in one of three ways:

1. It keeps trying the same approach and getting less out of it each time.
2. It finishes a piece of work, writes a summary, and then just sits there waiting.
3. The session closes or the context gets compacted, and the loop quietly stops with no error.

None of these are about the model being weak. They happen because nobody wired up the plumbing to keep the loop honest and alive. That plumbing is what this skill describes.

## What's inside

One skill — `deli-autoresearch`. It covers:

- The rule that, once started, the agent runs without stopping to ask questions
- Which files hold state and logs (`progress.json`, `findings.jsonl`, `directions_tried.json`, and so on)
- Prompts for the orchestrator, the workers, and the heartbeat watchdog
- How to tell the loop is stuck by counting results, not by asking the model how it's doing
- How to make each new attempt structurally different from the last
- How to schedule subagents, validate their output, and escalate when blocked

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

Then set up a task directory, write `task_spec.md`, start the orchestrator loop, and start a separate heartbeat watchdog alongside it. The skill body has the full prompts and file formats.

## License

MIT