# Deli AutoResearch

[English](./README.md) | 简体中文

一套让 agent 循环可以无人值守跑上几小时到几天的约定。这个仓库里没有代码——只有一个 skill，告诉 agent 怎么把状态写进文件、卡住时换个不一样的思路、以及循环死掉后怎么被拉起来。

## 要解决的问题

让 agent 自己跑久了，通常会在三种情况下出事：

1. 一直用同一个思路试，越试越没收获。
2. 做完一段工作，写完总结，就停在那等，再也不往下走。
3. 会话被关掉、上下文被压缩，循环就这么悄悄停了，还不报错。

这些都不是模型能力不行。是因为没人给它搭好让它老实跑下去的那套东西。这个 skill 讲的就是那套东西。

## 里面有什么

一个 skill —— `deli-autoresearch`。它写了：

- 一旦开始跑，就不再停下来问问题这条规矩
- 哪些文件存状态、哪些存日志（`progress.json`、`findings.jsonl`、`directions_tried.json` 这些）
- orchestrator、worker、heartbeat watchdog 三种角色的 prompt
- 怎么靠数结果判断循环是不是卡住了，而不是去问模型自己感觉怎么样
- 怎么让每一轮的尝试和前面那些在结构上不一样
- 怎么调度 subagent、怎么校验它们的产出、卡住了怎么上报

## 安装

直接装：

```text
/plugin install deli-autoresearch@deli-autoresearch
```

或者直接把 skill 复制到你的 skills 目录：

```text
skills/deli-autoresearch/SKILL.md
```

## 使用

要开始一个无人值守的长任务时，调用这个 skill：

```text
/skill deli-autoresearch
```

然后建一个任务目录，写好 `task_spec.md`，启动 orchestrator 循环，再单独起一个 heartbeat watchdog 跟着它。完整的 prompt 和文件格式见 skill 正文。

## 许可证

MIT
