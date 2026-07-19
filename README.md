# Relay Between Agents

**A safe, auditable relay between planning AIs and coding agents—without repetitive copy-pasting.**

Relay Between Agents 是一套轻量的多 Agent 施工控制面：规划型 AI 负责讨论需求、形成结构化工单和审计结果，coding agent 负责检查仓库、实现、测试并交付收据。两端通过受控的 Inbox、状态机与 Outbox 通信，让人类保留最终控制权，同时不再充当几千字 prompt 的剪贴板。

```text
Planner AI
  → authorized task
  → Relay Inbox
  → validator + approval policy
  → Foreman / Coding Agent
  → receipt + audit report
  → Relay Outbox
  → Planner AI
```

## 教程

- [完整中文教程：从零搭建一个安全、可审计的 Agent Relay](./docs/tutorial.zh-CN.md)

教程包含：

- Planner、Owner 与 Foreman 的职责边界；
- Google Sheet / Drive 传输层与最小权限设计；
- Task Packet schema、状态机与 owning-row；
- 幂等、冲突检测、KILL switch 和 crash recovery；
- `AUTO / ASK_OWNER / BLOCKED` 持久授权政策；
- Receipt、Outbox 与端到端回读验证；
- CSV/公式注入、secret 泄露和重复确认等常见陷阱；
- 一段可直接交给 coding agent 的实现 prompt。

## 设计原则

Relay 不是让两个 AI 无约束地互相聊天。它强调：

1. **人类拥有最终决定权**：高影响操作必须明确授权。
2. **工单有界**：每项任务都有 filescope、门禁与停止条件。
3. **施工可恢复**：状态、收据、哈希和回滚依据不依赖聊天记忆。
4. **默认最小权限**：不因自动化便利而扩张云端或机器权限。
5. **先稳定后常驻**：先验证手动 scan，再单独评估 watcher、daemon 或 webhook。

## Scope

这是一个隐私清洗后的参考架构，不包含任何真实身份 prompt、私人对话、transcript、机器路径、Drive ID、token 或生产仓库秘密。示例旨在教你如何为自己的工作流建立边界，而不是复制某个私人 AI 系统。

## License

[CC BY-NC-SA 4.0](./LICENSE.md)
