# Relay Between Agents — Architecture Notes

本文件只记录那些若不精确定义，就可能造成重复施工、越权、数据泄露或无法恢复的实现约束。产品介绍、人工挡位和从零搭建步骤请先读 [README.md](./README.md)。

## 1. 不变量

任何传输层和模型组合都必须保持：

1. 一个 `task_id` 只对应一个逻辑任务。
2. 同一 `idempotency_key` 不得产生第二次副作用。
3. 只有 `authorized` 任务可被认领。
4. Planner 声明的授权等级不能覆盖 Foreman 本机政策。
5. 状态更新必须精确命中 owning-row。
6. 高影响动作必须在副作用之前暂停。
7. KILL 对 scan、claim、execute、advance 和 watcher 全部生效。
8. crash/retry 后从持久状态恢复，不凭会话记忆猜测。
9. receipt 与 Outbox 报告必须可交叉校验。
10. 传输内容永远是不可信输入。

## 2. 建议 schema

```yaml
schema_version: 0
task_id: PROJECT-YYYYMMDD-NNN
status: authorized
updated_by: planner
idempotency_key: sha256:<canonical-packet>
from: planner
approval_level: auto
title: bounded title
base_expect: <git-sha-or-null>
filescope:
  - src/example/**
excluded:
  - credentials
gates:
  - targeted tests
stop_conditions:
  - base drift
body: |
  Observable change and acceptance criteria.
receipt_to: outbox
created_at: 2026-07-19T00:00:00Z
claim_token: null
lease_until: null
receipt_hash: null
receipt_at: null
error: null
```

### Canonicalization

生成 `idempotency_key` 前：

- 固定字段顺序；
- 字符串使用 UTF-8 与统一换行；
- 数组保留语义顺序，不随意排序；
- 排除运行时字段，如 status、lease、receipt；
- 禁止浮动时间戳进入逻辑任务指纹。

相同 `task_id` 不同 idempotency key 是冲突，不是新版任务。修改任务应产生显式 revision 或新的 task ID。

## 3. 状态机

```text
draft/proposed
      ↓ owner or delegated planner
authorized
      ↓ compare-and-set claim
active ─────────────→ awaiting_owner
  │                         │ approve / edit / reject
  │                         └──────────→ active | blocked
  ├────────→ complete
  ├────────→ blocked
  └────────→ expired
```

状态只能沿允许的边推进。任何跳跃都必须由显式迁移处理，不应靠“把单元格改成 complete”绕过。

### Owning-row compare-and-set

认领前同时匹配：

```text
task_id == expected
idempotency_key == expected
status == authorized
claim_token is empty
```

完成前同时匹配：

```text
task_id == expected
idempotency_key == expected
status == active
claim_token == this_worker
```

若任一条件不符，停止并重新读取。不要用标题、行号缓存或“第一个 active”做模糊更新。

Google Sheets 本身不是事务数据库。若并发会超过一个写执行者，建议改用带条件更新的数据库，或在 Sheet 前增加单写入协调器。

## 4. Claim、lease 与 crash recovery

Watcher 模式应写入：

- 随机 `claim_token`；
- `claimed_at`；
- 有界 `lease_until`；
- worker/session ID。

lease 过期不等于可以立即重做副作用。恢复者必须先查：

1. processed packet；
2. receipt；
3. git commit/tag；
4. deploy marker；
5. 外部系统幂等键。

只有证明副作用未发生，才重新执行；否则进入人工 reconcile。

## 5. 授权分类器

建议输出可解释结果：

```json
{
  "decision": "AUTO",
  "rule_id": "AUTO_BOUNDED_CODE_V1",
  "reasons": ["filescope bounded", "base matches", "no live action"],
  "packet_hash": "sha256:...",
  "policy_version": 1
}
```

分类顺序必须 fail-closed：

```text
KILL / schema failure / secret → BLOCKED
destructive / deploy / permission / external identity → ASK_OWNER
explicit allow rule + bounded scope → AUTO
anything unmatched → ASK_OWNER or BLOCKED
```

不要让模型自由解释正则后决定权限。模型可以提出分类，确定性规则负责最终门禁。

## 6. 输入安全

Sheet、Issue、MCP 参数和 Outbox 回执都视为不可信数据。

最低要求：

- 使用 RFC 4180 CSV parser 或官方 API；
- API 写 Sheet 时优先 RAW 值语义；
- 不 `eval`；
- 不执行单元格公式；
- 不把 cell 拼接进 shell；
- 拒绝或惰性化以 `=`, `+`, `-`, `@` 开头的外部字符串；
- shell 参数使用 argv/exec form，而非字符串命令；
- 路径 canonicalize 后再做 filescope 判断；
- 拒绝 `..`、符号链接逃逸和大小写绕过；
- secret 扫描只是补充，不能替代权限隔离。

Claude Code 官方文档明确提醒 hooks 以当前系统用户权限执行，因此 hook 输入必须验证，敏感路径必须排除。不要把 hook 当安全沙箱。

## 7. Filescope 与冲突检测

将 glob 转成规范化路径集合，至少检测：

- exact file overlap；
- parent/child overlap；
- generated artifact 与 source 的关联；
- migration/schema 与所有消费者的隐式冲突；
- live launcher、lockfile、共享配置等全局热点。

并发建议：一个写任务 + 一个只读评审。任何不确定交集按冲突处理并串行。

## 8. KILL

最小 KILL 可以是仅 Owner 可删除的文件：

```text
~/relay/KILL
```

每个入口在产生副作用前检查；长期任务也应在阶段边界复查。KILL 存在时只允许：

- 查询状态；
- 写一条 KILL receipt；
- 安全释放本任务持有的临时锁（不得推进业务状态）。

Agent 不得自行删除、忽略或“临时绕过” KILL。

## 9. Transport adapter

把 Sheet、GitHub Issues 或数据库封装为同一接口：

```ts
interface RelayTransport {
  listAuthorized(): Promise<Task[]>;
  claim(taskId: string, key: string, token: string): Promise<boolean>;
  requestApproval(taskId: string, payload: ApprovalCard): Promise<void>;
  complete(taskId: string, token: string, receipt: Receipt): Promise<void>;
  block(taskId: string, token: string, error: BoundedError): Promise<void>;
}
```

Executor adapter 同样隔离：

```ts
interface Executor {
  run(task: ValidatedTask, limits: RunLimits): AsyncIterable<RunEvent>;
  cancel(runId: string): Promise<void>;
}
```

这样可以把 ChatGPT + CC 替换成自建前端 + API，而不重写状态与授权逻辑。

## 10. Watcher

Watcher 只负责唤醒，不负责降低门禁。

必须具备：

- 单实例锁或 leader election；
- 带 jitter 的轮询和失败退避；
- 每批认领上限；
- 并发上限；
- 每任务 timeout、turn limit、cost budget；
- lease 与 crash recovery；
- 结构化日志和 bounded error；
- KILL；
- 重启后幂等恢复；
- secret 不进入 argv、日志或 packet。

Claude Code hooks 可用于生命周期检查或通知，但 hook 会以本机用户权限运行。若要自动启动完整 agent，优先使用受限的非交互 CLI/Agent SDK 调用，并显式设置 allowed tools 与运行上限。

## 11. Approval interrupt

暂停必须发生在副作用之前，并持久化：

```json
{
  "task_id": "DEMO-001",
  "status": "awaiting_owner",
  "action": "deploy production",
  "impact": "restart one service",
  "rollback": "restore tag pre-demo and restart",
  "approval_nonce": "random-one-time-value",
  "expires_at": "..."
}
```

批准时同时匹配 task ID、action hash、nonce、状态和有效期。工单被编辑后旧批准自动失效。

LangGraph 等框架提供可持久化 interrupt/resume；没有框架时，任务存储中的 `awaiting_owner` + nonce 就能实现同样语义。

## 12. Receipt

机器 receipt 示例：

```json
{
  "task_id": "DEMO-001",
  "idempotency_key": "sha256:...",
  "status": "complete",
  "classification": "AUTO",
  "rule_id": "AUTO_BOUNDED_CODE_V1",
  "policy_version": 1,
  "base_before": "0123abc",
  "head_after": "4567def",
  "tests": ["unit: pass", "typecheck: pass"],
  "files_changed": ["src/status/render.ts"],
  "live_touched": false,
  "receipt_sha256": "...",
  "completed_at": "2026-07-19T12:34:56Z"
}
```

人类报告与 receipt 共享 task ID 和哈希。上传时使用不可覆盖语义；Planner 回读后比对 `receipt_hash`。

## 13. 测试矩阵

最低端到端测试：

1. authorized/AUTO 只读任务成功；
2. proposed 不被消费；
3. ASK_OWNER 副作用前暂停；
4. 拒绝后 blocked；
5. 编辑工单使旧批准失效；
6. KILL 对 scan/claim/execute/advance/watcher 全拒；
7. 重复 task ID 拒绝；
8. 重复 idempotency key 拒绝；
9. 同 task ID 不同 key 作为冲突；
10. stale base 阻塞；
11. filescope 冲突阻塞；
12. 路径穿越与符号链接逃逸拒绝；
13. CSV 逗号、引号、换行惰性往返；
14. 公式样式字符串不执行；
15. secret fixture 被拒收且不进入日志；
16. claim 竞争只有一个成功；
17. claim 后 crash 不重复副作用；
18. lease 过期进入 reconcile；
19. receipt 只更新 owning-row；
20. Outbox 可被 Planner 精确读取并核对哈希；
21. watcher 重启不重复执行；
22. timeout、turn limit、cost budget 生效；
23. 授权政策未知规则 fail-closed；
24. live/deploy 路径在无批准时不可达。

## 14. 参考资料

- [Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in)
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview)
- [Google Drive API scopes](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- [Google Sheets values.update](https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/update)
- [rclone Google Drive backend](https://rclone.org/drive/)
- [LangGraph interrupts](https://docs.langchain.com/oss/javascript/langgraph/interrupts)
- [GitHub deployment protection rules](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
