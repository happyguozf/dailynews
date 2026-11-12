---
title: "Hermes Agent v0.16 Kanban Swarm：让多 Agent 协作从聊天变成可追踪的任务图"
date: 2026-11-12
tags: [AI Agent, Hermes Agent, Kanban, Multi-Agent]
author: "happyguozf"
---

设想凌晨两点，一项架构调研还没结束：三个 AI Agent 分别研究故障、设计方案和制定运维计划，其中一个中途退出。第二天接手的人如何知道哪些结果可信、谁还没完成，以及最终报告能否开始写？把任务都塞进同一段聊天，并不容易回答这些问题。

Hermes Agent 的 Kanban Swarm 给出的思路是：**用持久化任务图管理协作，用明确依赖控制交接**。本文讨论的是 Hermes Agent **v0.16.0**，官方 Git 标签为 `v2026.6.5`；历史实现固定在提交 `3c231eb3979ab9c57d5cd6d02f1d577a3b718b43`，而不是把今天在线文档里的所有功能套回旧版本。[1]、[2]

## 从并行对话走向可恢复的工作流

普通多 Agent 委派很适合短时、一次性的并行任务；当任务跨越多个阶段，需要等待依赖、处理失败或交给其他角色继续时，仅靠对话记录就会增加协调负担。Kanban 的价值不是让模型突然更聪明，而是把**谁做、何时做、交付什么、失败在哪里**变成可检查的状态。

在 v0.16 的实现里，Swarm 是建立在既有 Kanban kernel 上的轻量拓扑辅助层，并没有另起一套调度器。[2] 这意味着系统复用已有任务、事件和调度机制，而不是把并行调用包装成一个无法追溯的黑盒。

## 架构：任务图是骨架，blackboard 是共享上下文

一个典型 Swarm 的依赖形状如下：[2]

```text
planning root（创建后完成）
        │
        ├── worker A ──┐
        ├── worker B ──┼── verifier ── synthesizer
        └── worker C ──┘
```

这里的箭头表示依赖，而非即时消息流。root 保存目标与拓扑；多个 specialist worker 可以并行；verifier 等待全部 worker；synthesizer 再等待 verifier。共享 blackboard 通过 root 任务上的结构化 JSON 评论实现，相同 key 的后续值覆盖前值；worker 还通过任务 handoff 传递结果。[2]

底层协调数据存入 SQLite，gateway 中的 dispatcher 负责领取可运行任务并启动对应 profile 的 worker 进程；workspace 则为任务提供工作目录。[4]、[5] **为什么重要？**任务状态和交接信息不必只存在于某个 agent 的上下文窗口，接手者可以检查已有记录；但共享评论不自动保证事实正确，也不替代对产物的验证。

## 状态与依赖：完成了，不等于通过了质量审核

常见路径是 `todo → ready → running → done`；源码还定义了 `triage`、`blocked`、`review`、`scheduled` 和 `archived` 等状态。[5] 不要把这些状态名误读为所有版本都有相同的操作入口。依赖环和自依赖会被拒绝；在固定版实现中，依赖晋级把父任务的 `done` **或 `archived`** 都视为满足。[5]

**为什么重要？**如果把归档误当成“未通过”，就可能低估后续任务被放行的可能性。状态反映的是调度条件，不是业务质量证明；归档策略应与团队的验收流程一致。

verifier 的细节尤其值得留意：其任务正文要求证据充分时完成，并附上 `{"gate":"pass"}`；否则应 block。[2] 但固定版依赖晋级依据父任务状态，并不强制检查该 metadata。[5] 上游研究在隔离 SQLite 探针中观察到：即使 verifier 完成时没有 gate metadata，synthesizer 仍会变为 `ready`。这只是 kernel 级探针，并非真实模型或 gateway 的端到端测试。

因此，verifier 是**需要执行者遵守的工作协议，而非数据库强制的内容质量门**。生产流程中若必须验证测试结果或审批记录，应在外部系统实施可审计的硬检查；不要仅凭一个 metadata 字段或任务状态就放行高风险操作。

## 实践：创建一个多区域故障演练任务图

以下命令针对上述固定版源码编写，假设已安装该版本并配置 provider、凭据和可运行的 gateway；`researcher`、`architect`、`sre`、`reviewer`、`writer` 必须是已经存在且可调度的 profile。先在隔离测试板运行：创建任务可能触发真实模型调用并产生费用。[3]、[4]

```bash
hermes kanban swarm "Design a multi-region failover plan" \
  --worker "researcher:Research failure scenarios" \
  --worker "architect:Design the architecture" \
  --worker "sre:Develop the operations plan" \
  --verifier reviewer \
  --synthesizer writer \
  --json
```

每个 `--worker` 使用 `PROFILE:TITLE`；固定版 parser 接受重复的 `--worker`，而同版文档中的 `--workers` 示例与实现不一致，应以所安装版本的 `--help` 和源码为准。[3]、[4] 命令创建的是任务图，不代表所有 worker 已执行，更不代表输出已获验证。运行后检查返回的 task ID、各卡状态、失败原因和交付物；需要停止自动执行时，应由操作者按部署的 gateway 运维流程处理，而不是假设创建命令自带停止机制。

**为什么重要？**把 profile 存在、调度器可用和输出验收分成不同前提，才能区分“图已建好”和“工作已完成”。本文没有运行上述真实模型端到端示例，也不声称得到过演练方案。

## 可靠性与安全：隔离工作目录不是安全沙箱

固定版 dispatcher 会处理过期或崩溃的领取、重算就绪状态并派发任务；重试达到限制后可阻塞任务。心跳和运行时限有助于发现卡住的工作。[4]、[5] **为什么重要？**恢复机制降低了人工追踪中断的负担，但一次任务可能重试，外部写入并没有因此获得 exactly-once 保证。发送消息、部署或创建资源时，应由目标系统提供幂等键，并在危险动作前增加明确审批。

`scratch`、`dir` 和 Git `worktree` 提供不同的工作目录组织方式，独立 profile 和进程也有助于职责分工。[4]、[5] 它们**不等于操作系统级安全沙箱**：凭据、文件权限和网络访问仍须按最小权限配置；并发修改同一文件也仍可能冲突。特别是依赖关系只决定何时开始，不会自动解决代码合并或证明结果正确。

## 版本边界与适用场景

当前在线 Kanban 文档持续演进；其中原子建图和同卡 review 等描述，不应直接当作 v0.16 的保证。固定版 Swarm 创建图时依序调用任务创建、完成和评论操作；其 verifier 也是一张独立依赖卡，而不是自动等同于当前的同卡 review。[2]、[6] 因此，若创建过程报错或中断，应先检查板上是否留下不完整任务图，再决定如何清理或重试；不要假定整张图已回滚。这一区分很实际：升级后照搬旧命令，或者拿新文档反推旧版行为，都可能让运维预期落空。

Swarm 适合可拆分为独立研究、验证和汇总阶段的工作，例如方案评估、文档整理或带明确验收条件的开发任务；对一步即可完成的小任务，它可能只增加调度和模型成本。涉及部署、财务或敏感数据的流程，则必须另设权限边界、人工审批与可验证的外部控制。

把 Agent 组织成团队，难点不只是同时启动多少个模型，而是让每个交接点都能被解释、检查和恢复。Kanban Swarm 提供了任务图和运行机制；真正可靠的协作，还取决于你如何设计验收、隔离与责任边界。

## Sources

1. [v0.16.0 release](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5)
2. [固定版 Swarm 源码](https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban_swarm.py)
3. [固定版 Kanban CLI 源码](https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban.py)
4. [固定版 Kanban 文档](https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/website/docs/user-guide/features/kanban.md)
5. [固定版 Kanban 数据库源码](https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban_db.py)
6. [当前在线 Kanban 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)

[1]: https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5
[2]: https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban_swarm.py
[3]: https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban.py
[4]: https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/website/docs/user-guide/features/kanban.md
[5]: https://github.com/NousResearch/hermes-agent/blob/3c231eb3979ab9c57d5cd6d02f1d577a3b718b43/hermes_cli/kanban_db.py
[6]: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
