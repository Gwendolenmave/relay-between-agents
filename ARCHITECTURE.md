# Relay Between Agents — Architecture & Implementation Guide

本文件是给**第一次接手 Relay 施工的小机和工程师**的实施手册。产品目的、适用人群和人工挡位请先读 [README.md](./README.md)。

这里不追求绑定某个模型或云服务，而是把容易造成重复施工、权限越界、数据泄露和不可恢复状态的实现细节写死。示例以 TypeScript/Node.js 为主；换成 Python、Go 或其他语言时，请保留相同不变量。

> 先把手动 scan 的闭环做对，再实现 watcher。先把只读任务做对，再开放写文件。先把 staging 做对，再考虑 live deploy。

---

## 0. 第一次接手时，先做什么？

不要一上来写代码。先只读回答：

1. Planner、Foreman、Owner 分别是谁？
2. Inbox 与 Outbox 目前是什么载体？
3. 项目的 canonical repo、branch、HEAD 和 live process 是什么？
4. 是否已经存在 Relay、ACTIVE-WORK、policy、KILL 或旧 watcher？
5. 当前人工挡位是 0–4 中哪一档？
6. 哪些动作已被 Owner 委托？哪些永远必须问？
7. 是否有可恢复的本地与 off-device 备份？
8. 这一轮是否被授权创建 daemon、OAuth、端口、webhook 或新依赖？

若任何答案不清楚，交付一页现状报告并 STOP。不要用“看起来应该是”补齐权限事实。

### 开工前门禁

```text
[ ] KILL 不存在
[ ] 唯一账本已定位
[ ] canonical HEAD 与工单 base_expect 一致
[ ] working tree 状态已记录
[ ] active task 的 filescope 已加载
[ ] 本机 policy 已加载并有版本号
[ ] Inbox/Outbox 双向探针已通过
[ ] secret 不会进入 packet、argv、日志或报告
[ ] 回滚与恢复路径存在
[ ] 本轮授权覆盖将要做的动作
```

任何一项失败都不是“先做再说”，而是 `BLOCKED`。

---

## 1. 参考实现与版本策略

参考栈：

```text
Planner: ChatGPT official app
Task Store: Google Sheet
Outbox: Google Drive folder
Foreman: local Node.js control process
Executor: Claude Code / Claude Agent SDK
Repository: Git
Local state: files + one canonical ACTIVE-WORK ledger
Transport bridge: Sheets API or rclone
```

推荐 Node.js 20+。示例依赖：

```json
{
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "<pin-a-reviewed-version>",
    "csv-parse": "<pin-a-reviewed-version>",
    "picomatch": "<pin-a-reviewed-version>",
    "zod": "<pin-a-reviewed-version>"
  }
}
```

不要在教程里写一个“永远最新”的版本号。实施时：

1. 读取官方文档与当前 lockfile；
2. 选择并锁定已评审版本；
3. 把版本、安装来源和 checksum 记入报告；
4. 升级 SDK 时重新跑权限与 crash/retry 测试。

### 不要混用两种控制层

以下两条任选其一作为主控制面：

- **本地 Foreman 脚本控制 SDK**：适合自动 watcher；
- **交互式 Claude Code 会话读取 Relay**：适合人工叫醒。

可以共享 packet、policy 和 receipts，但不能让两者同时认领同一 Inbox。否则必须引入真正的分布式 claim 存储。

---

## 2. 威胁模型与信任边界

Relay 至少有五个不同信任域：

```text
Owner input                  trusted for approval, not for syntax
Planner output               untrusted task proposal
Sheet / Issue / MCP payload  untrusted transport data
Foreman policy + validator   trusted control plane
Executor/model output        untrusted proposed actions
OS / git / cloud APIs        side-effect boundary
```

即使 Planner 是你熟悉的 AI，它写进表里的内容仍按不可信输入处理。原因包括：

- 模型误判；
- prompt injection；
- connector 或第三方文件被污染；
- CSV/公式注入；
- 旧任务重放；
- 人类误改行；
- 会话被压缩后丢失上下文。

### 核心不变量

1. 一个 `task_id` 只代表一个逻辑任务。
2. 同一 `idempotency_key` 不产生第二次副作用。
3. 只有 `authorized` 任务可以被认领。
4. Planner 声明的 `auto` 不能覆盖 Foreman 本机 policy。
5. 状态只更新任务自己的 owning-row。
6. 高影响动作在副作用**之前**暂停。
7. KILL 对 scan、claim、execute、advance 和 watcher 全部生效。
8. crash/retry 从持久状态恢复，不凭聊天记忆猜。
9. receipt、报告和 task row 能通过 ID 与哈希交叉验证。
10. 传输内容永远不直接变成 shell、SQL、路径或权限决定。

---

## 3. 推荐目录结构与权限

```text
~/relay/
├── bin/                     # 可执行入口；只存代码
│   ├── scan.mjs
│   ├── run.mjs
│   ├── approve.mjs
│   └── doctor.mjs
├── config/
│   ├── policy.json          # 持久授权政策
│   ├── transport.json       # 仅非秘密配置
│   └── projects.json        # project → canonical path 映射
├── inbox/                   # 已下载但尚未验证
├── validated/               # canonical packet
├── processed/               # 终态 packet
├── approvals/               # 待批准动作与一次性 nonce
├── receipts/                # 机器可读 JSON
├── outbox/                  # 人类可读 Markdown 报告
├── locks/                   # 单写者锁
├── logs/                    # 脱敏结构化日志
├── state/                   # cursor、lease、worker identity
├── README.md                # 操作手册
└── KILL                     # 默认不存在
```

建议：

- Relay 根目录、receipts、approvals 和配置使用仅当前用户可读写权限；
- OAuth token 使用系统 secret store 或权限严格的独立配置文件；
- token 路径不进入 git；
- Outbox 默认不包含私人正文；
- canonical repo 不存 Relay 的运行秘密；
- KILL 的删除权只属于 Owner 使用的独立可信入口。

**坑：** 不要把 KILL 放在会被 build、清理脚本或 `git clean` 自动删除的目录里。

---

## 4. Task schema：先结构化，再让模型解释

推荐把“动作类型”显式列出来，不要只靠自然语言 body 推断风险。

```yaml
schema_version: 1
task_id: PROJECT-20260719-001
revision: 0
status: authorized
updated_by: planner
idempotency_key: sha256:<canonical-logical-task>
project_id: demo-project
from: planner
approval_level: auto
title: Add bounded status field
requested_actions:
  - read
  - edit
  - test
  - commit
base_expect: 0123abc
filescope:
  - src/status/**
  - tests/status/**
excluded:
  - .env
  - data/private/**
gates:
  - targeted-tests
  - typecheck
stop_conditions:
  - base-drift
  - filescope-conflict
  - permission-expansion
body: |
  Observable behavior and acceptance criteria.
receipt_to: outbox
created_at: 2026-07-19T00:00:00Z
claim_token: null
claimed_at: null
lease_until: null
approval_id: null
receipt_hash: null
receipt_at: null
error_code: null
error_summary: null
```

### TypeScript schema 示例

```ts
// schema.ts
import { z } from "zod";

export const Status = z.enum([
  "proposed",
  "authorized",
  "active",
  "awaiting_owner",
  "complete",
  "blocked",
  "expired",
]);

export const Action = z.enum([
  "read",
  "audit",
  "edit",
  "test",
  "build",
  "commit",
  "tag",
  "rebase",
  "merge_ff",
  "deploy",
  "restart_live",
  "delete",
  "force",
  "history_rewrite",
  "migration",
  "oauth",
  "permission_change",
  "spend",
  "daemon",
  "webhook",
  "external_publish",
  "private_memory",
]);

const BoundedText = z.string().max(20_000);
const PathPattern = z.string()
  .min(1)
  .max(500)
  .refine((s) => !s.includes("\0"), "NUL is forbidden")
  .refine((s) => !s.split(/[\\/]/).includes(".."), "parent traversal is forbidden");

export const LogicalTaskSchema = z.object({
  schema_version: z.literal(1),
  task_id: z.string().regex(/^[A-Z0-9][A-Z0-9_-]{2,79}$/),
  revision: z.number().int().min(0),
  project_id: z.string().regex(/^[a-z0-9][a-z0-9_-]{1,63}$/),
  from: z.enum(["planner", "owner"]),
  approval_level: z.enum(["auto", "ask_owner", "design_only"]),
  title: z.string().min(1).max(200),
  requested_actions: z.array(Action).min(1).max(30),
  base_expect: z.string().regex(/^[0-9a-f]{7,64}$/).nullable(),
  filescope: z.array(PathPattern).max(100),
  excluded: z.array(PathPattern).max(100),
  gates: z.array(z.string().min(1).max(100)).max(50),
  stop_conditions: z.array(z.string().min(1).max(200)).max(50),
  body: BoundedText,
  receipt_to: z.enum(["outbox", "outbox+owner"]),
  created_at: z.string().datetime(),
}).strict();

export const TaskRowSchema = LogicalTaskSchema.extend({
  status: Status,
  updated_by: z.enum(["planner", "foreman", "owner"]),
  idempotency_key: z.string().regex(/^sha256:[0-9a-f]{64}$/),
  claim_token: z.string().uuid().nullable(),
  claimed_at: z.string().datetime().nullable(),
  lease_until: z.string().datetime().nullable(),
  approval_id: z.string().uuid().nullable(),
  receipt_hash: z.string().regex(/^sha256:[0-9a-f]{64}$/).nullable(),
  receipt_at: z.string().datetime().nullable(),
  error_code: z.string().max(80).nullable(),
  error_summary: z.string().max(500).nullable(),
}).strict();

export type LogicalTask = z.infer<typeof LogicalTaskSchema>;
export type TaskRow = z.infer<typeof TaskRowSchema>;
```

### 为什么 `.strict()` 很重要？

未知字段可能是新版本、拼写错误或注入内容。静默忽略会让 Planner 以为某个限制生效，而 Foreman 根本没读到。遇到未知字段应阻塞并要求 schema migration。

### Sheet 单元格中的数组

如果 Sheet 只能存字符串，将数组存成 JSON：

```ts
export function parseJsonArrayCell(name: string, raw: unknown): string[] {
  if (typeof raw !== "string") throw new Error(`${name}: expected JSON string`);
  let value: unknown;
  try {
    value = JSON.parse(raw);
  } catch {
    throw new Error(`${name}: invalid JSON`);
  }
  return z.array(z.string()).max(100).parse(value);
}
```

**错误写法：** 用逗号 `split(',')` 解析 paths。文件名、glob 或说明本身可能包含逗号。

---

## 5. Sheet/CSV 输入：防公式、防错列、防悄悄截断

### 推荐列所有权

将列按写入者分区：

```text
Planner-owned: task identity, body, actions, scope, gates
Foreman-owned: status, claim, lease, receipt, error
Owner-owned: approval decision, approval nonce
```

能用 Sheet protected ranges 就保护；不能保护也必须在代码里只更新自己负责的列。Foreman 绝不能整行覆盖，因为那会抹掉 Planner 或 Owner 在并发期间写入的值。

### 读取公式而不是计算结果

使用 Sheets API 时，读取任务区应使用 `valueRenderOption=FORMULA`，这样 `=IMPORTDATA(...)` 等公式不会伪装成普通计算结果。写入使用 `valueInputOption=RAW`，防止以 `=` 开头的报告被执行成公式。

```ts
function rejectFormulaLike(value: unknown, field: string): void {
  if (typeof value !== "string") return;
  if (/^[\s\u0000-\u001f]*[=+\-@]/u.test(value)) {
    throw new Error(`${field}: formula-like input is forbidden`);
  }
}
```

注意：负数等合法值若以字符串承载会被这条规则拒绝。解决办法是让数字字段使用真正 number 类型，而不是为了方便放宽所有字符串。

### 正确解析 RFC 4180 CSV

```ts
// csv.ts
import { parse } from "csv-parse/sync";

const EXPECTED_HEADERS = [
  "schema_version", "task_id", "revision", "status", "updated_by",
  "idempotency_key", "project_id", "from", "approval_level", "title",
  "requested_actions", "base_expect", "filescope", "excluded", "gates",
  "stop_conditions", "body", "receipt_to", "created_at", "claim_token",
  "claimed_at", "lease_until", "approval_id", "receipt_hash", "receipt_at",
  "error_code", "error_summary",
] as const;

export function parseRelayCsv(csv: string): Record<string, string>[] {
  const rows = parse(csv, {
    bom: true,
    columns: false,
    skip_empty_lines: true,
    relax_column_count: false,
    relax_quotes: false,
  }) as string[][];

  if (rows.length === 0) throw new Error("empty CSV");
  const header = rows[0];
  if (new Set(header).size !== header.length) throw new Error("duplicate header");
  if (header.length !== EXPECTED_HEADERS.length) throw new Error("header width drift");
  EXPECTED_HEADERS.forEach((name, i) => {
    if (header[i] !== name) throw new Error(`header[${i}] expected ${name}`);
  });

  return rows.slice(1).map((cells, rowOffset) => {
    if (cells.length !== header.length) {
      throw new Error(`row ${rowOffset + 2}: width mismatch`);
    }
    return Object.fromEntries(header.map((name, i) => [name, cells[i]]));
  });
}
```

**绝对不要：**

```bash
# WRONG: quotes/newlines break; content may become code
cut -d, -f17 relay.csv
eval "$(cut -d, -f17 relay.csv)"
```

### rclone CSV 的额外限制

rclone 适合做轻量桥接，但 CSV 导出/导入不是事务数据库：

- 导出时确认 header、引号、换行、空值逐格一致；
- import/export format 必须显式设置；
- 只允许一个 Foreman 写者；
- 不要整表下载、修改、整表覆盖作为常规状态更新；
- 若需要多个并发 worker，迁移到带条件更新的数据库或单写协调服务。

---

## 6. Canonicalization 与幂等键

幂等键只覆盖**逻辑任务**，不能包含 status、claim、lease、receipt 等运行字段。

```ts
// canonical.ts
import { createHash } from "node:crypto";
import type { LogicalTask } from "./schema.js";

function canonical(value: unknown): unknown {
  if (value === null || typeof value === "string" || typeof value === "boolean") {
    return value;
  }
  if (typeof value === "number") {
    if (!Number.isFinite(value)) throw new Error("non-finite number");
    return value;
  }
  if (Array.isArray(value)) return value.map(canonical); // array order is semantic
  if (typeof value === "object") {
    const entries = Object.entries(value as Record<string, unknown>)
      .sort(([a], [b]) => a.localeCompare(b, "en"))
      .map(([k, v]) => [k, canonical(v)]);
    return Object.fromEntries(entries);
  }
  throw new Error(`unsupported canonical type: ${typeof value}`);
}

export function canonicalJson(value: unknown): string {
  return JSON.stringify(canonical(value));
}

export function logicalTaskKey(task: LogicalTask): string {
  const bytes = Buffer.from(canonicalJson(task), "utf8");
  return `sha256:${createHash("sha256").update(bytes).digest("hex")}`;
}
```

规则：

- 同一 `task_id` + 同一 key：重复投递，安全忽略并回已有 receipt；
- 同一 `task_id` + 不同 key：冲突，BLOCKED；
- 不同 `task_id` + 同一 key：疑似重复任务，默认 BLOCKED；
- 修改任务：增加 revision，并重新授权；不要悄悄原地改 body。

**坑：** 把 `created_at` 自动设为“现在”后再算 key，会让同一任务每次重试都变成新任务。时间戳应由 Planner 固定，或从逻辑 hash 中排除。

---

## 7. 状态机：所有推进都要显式合法

```ts
// state.ts
import type { TaskRow } from "./schema.js";

type Status = TaskRow["status"];

const ALLOWED: Record<Status, readonly Status[]> = {
  proposed: ["authorized", "blocked", "expired"],
  authorized: ["active", "blocked", "expired"],
  active: ["awaiting_owner", "complete", "blocked", "expired"],
  awaiting_owner: ["active", "blocked", "expired"],
  complete: [],
  blocked: [],
  expired: [],
};

export function assertTransition(from: Status, to: Status): void {
  if (!ALLOWED[from].includes(to)) {
    throw new Error(`illegal transition ${from} -> ${to}`);
  }
}
```

终态默认不可重开。要重试 blocked 任务，创建新的 revision/task ID，并保留旧 receipt；不要把 blocked 单元格手工改回 authorized 后丢失事故记录。

### 每次状态推进至少写什么？

```text
task_id
idempotency_key
old_status
new_status
updated_by
updated_at
claim_token（active 阶段）
policy_version
bounded reason
```

---

## 8. 单写者锁：先挡住同机重入

Google Sheet 不提供通用的 compare-and-set 行更新。最小实现必须先确保同一台机器只有一个 Foreman 写者。

```ts
// lock.ts
import { mkdir, rm, writeFile } from "node:fs/promises";
import path from "node:path";

export async function withProcessLock<T>(
  lockRoot: string,
  name: string,
  fn: () => Promise<T>,
): Promise<T> {
  const lockDir = path.join(lockRoot, `${name}.lock`);
  try {
    await mkdir(lockDir); // atomic: EEXIST means another owner
  } catch (error: any) {
    if (error?.code === "EEXIST") throw new Error(`lock busy: ${name}`);
    throw error;
  }

  await writeFile(
    path.join(lockDir, "owner.json"),
    JSON.stringify({ pid: process.pid, started_at: new Date().toISOString() }),
    { flag: "wx", mode: 0o600 },
  );

  try {
    return await fn();
  } finally {
    await rm(lockDir, { recursive: true });
  }
}
```

### stale lock 怎么办？

不要看到时间旧就自动删。先检查：

- PID 是否存在；
- worker 是否仍持有 task lease；
- receipt 或副作用是否已产生；
- 是否有第二个 namespace/WSL/container 中的同名 worker。

无法证明 owner 已死亡时，BLOCKED 等 Owner 处理。

---

## 9. Owning-row：Sheet 上的“近似 CAS”

在单写者前提下，使用“读 → 验证 → 写自己列 → 回读验证”。

```ts
type RowRef = { sheet: string; rowNumber: number; taskId: string; key: string };

interface SheetTransport {
  readRow(ref: RowRef): Promise<Record<string, unknown>>;
  updateForemanColumns(
    ref: RowRef,
    values: Record<string, string | number | null>,
    inputMode: "RAW",
  ): Promise<void>;
}

export async function advanceOwnedRow(
  sheet: SheetTransport,
  ref: RowRef,
  expectedStatus: string,
  claimToken: string | null,
  patch: Record<string, string | number | null>,
): Promise<void> {
  const before = await sheet.readRow(ref);
  if (before.task_id !== ref.taskId) throw new Error("row identity drift");
  if (before.idempotency_key !== ref.key) throw new Error("row key drift");
  if (before.status !== expectedStatus) throw new Error("row status drift");
  if ((before.claim_token ?? null) !== claimToken) throw new Error("claim drift");

  await sheet.updateForemanColumns(ref, patch, "RAW");

  const after = await sheet.readRow(ref);
  for (const [key, value] of Object.entries(patch)) {
    if ((after[key] ?? null) !== value) throw new Error(`write verification failed: ${key}`);
  }
}
```

这不是跨客户端强事务。Google Sheets `batchUpdate` 可以保证同一请求中的多个子更新一起成功或一起失败，但协作者仍可能在请求后改值。因此：

- 单 Foreman 写者；
- Planner 不写 Foreman-owned columns；
- Owner approval 使用独立列；
- 写后回读；
- 多 worker 时换数据库或单写服务。

**坑：** 行号不是长期身份。排序、插入和过滤会改变行号。每次操作前按 `task_id + idempotency_key` 重新定位，并验证只命中一行。

---

## 10. Claim、lease 与崩溃恢复

认领时生成随机 token：

```ts
import { randomUUID } from "node:crypto";

const claimToken = randomUUID();
const claimedAt = new Date();
const leaseUntil = new Date(claimedAt.getTime() + 15 * 60_000);
```

一次认领写入：

```text
status = active
claim_token = UUID
claimed_at = ISO time
lease_until = ISO time
updated_by = foreman
```

### lease 过期不等于“可以重做”

恢复者必须依序检查：

1. processed packet；
2. receipt；
3. git commit/tag；
4. build artifact hash；
5. deploy marker；
6. 外部 API 的 idempotency key；
7. live process 与目标版本。

只有能证明副作用未发生，才从安全 checkpoint 重试。否则进入 `reconcile_required`（可映射为 blocked + error code），让 Owner 决定。

**错误模式：** watcher 启动时把所有超时 active 行直接改回 authorized。这会重复 commit、发消息、部署或删除。

---

## 11. 本机授权分类器：确定性优先

Planner 的 `approval_level` 是请求，不是权限事实。最终结果由本机 policy 决定。

```ts
// policy.ts
type Decision = "AUTO" | "ASK_OWNER" | "BLOCKED";

const AUTO_ACTIONS = new Set([
  "read", "audit", "edit", "test", "build", "commit", "tag", "merge_ff",
]);

const ASK_ACTIONS = new Set([
  "deploy", "restart_live", "delete", "force", "history_rewrite", "migration",
  "oauth", "permission_change", "spend", "daemon", "webhook",
  "external_publish", "private_memory",
]);

export function classify(input: {
  kill: boolean;
  schemaOk: boolean;
  baseMatches: boolean;
  scopeConflict: boolean;
  secretsDetected: boolean;
  actions: string[];
  filescope: string[];
}): { decision: Decision; ruleId: string; reasons: string[] } {
  if (input.kill) return { decision: "BLOCKED", ruleId: "KILL_V1", reasons: ["KILL present"] };
  if (!input.schemaOk) return { decision: "BLOCKED", ruleId: "SCHEMA_V1", reasons: ["schema invalid"] };
  if (!input.baseMatches) return { decision: "BLOCKED", ruleId: "BASE_V1", reasons: ["base drift"] };
  if (input.scopeConflict) return { decision: "BLOCKED", ruleId: "CONFLICT_V1", reasons: ["filescope conflict"] };
  if (input.secretsDetected) return { decision: "BLOCKED", ruleId: "SECRET_V1", reasons: ["secret-like content"] };
  if (input.actions.some((a) => ASK_ACTIONS.has(a))) {
    return { decision: "ASK_OWNER", ruleId: "HIGH_IMPACT_V1", reasons: ["high-impact action"] };
  }
  if (input.filescope.length === 0 && input.actions.some((a) => a === "edit")) {
    return { decision: "BLOCKED", ruleId: "EMPTY_SCOPE_V1", reasons: ["edit without scope"] };
  }
  if (input.actions.every((a) => AUTO_ACTIONS.has(a))) {
    return { decision: "AUTO", ruleId: "BOUNDED_AUTO_V1", reasons: ["all actions delegated"] };
  }
  return { decision: "ASK_OWNER", ruleId: "UNKNOWN_ACTION_V1", reasons: ["unmatched action"] };
}
```

自然语言扫描只能**提高**风险，不能降低风险。例如 body 提到“顺便重启服务”，即使 `requested_actions` 漏写 restart，也应升级 ASK_OWNER；但 body 说“这是安全的”不能把 deploy 降成 AUTO。

每个 receipt 写：

```text
decision
rule_id
policy_version
reasons
packet_hash
```

---

## 12. Filescope：字符串前缀检查不够

错误写法：

```ts
// WRONG: /repo-evil starts with /repo
if (candidate.startsWith(repoRoot)) allow();
```

正确方向：

1. 固定 canonical repo root；
2. 对现有路径取 realpath；
3. 对新路径解析最近存在的 parent，检查 symlink；
4. 用 `path.relative` 判断是否逃逸；
5. 转成 repo-relative POSIX path 后匹配 glob；
6. excluded 永远优先于 filescope；
7. 施工后再次检查实际 diff。

```ts
// scope.ts
import fs from "node:fs/promises";
import path from "node:path";
import picomatch from "picomatch";

function isContained(root: string, candidate: string): boolean {
  const rel = path.relative(root, candidate);
  return rel === "" || (!rel.startsWith(`..${path.sep}`) && rel !== ".." && !path.isAbsolute(rel));
}

async function nearestExistingParent(input: string): Promise<string> {
  let current = input;
  for (;;) {
    try {
      await fs.lstat(current);
      return current;
    } catch (error: any) {
      if (error?.code !== "ENOENT") throw error;
      const parent = path.dirname(current);
      if (parent === current) throw new Error("no existing parent");
      current = parent;
    }
  }
}

export async function assertPathAllowed(
  repoRootInput: string,
  candidateInput: string,
  include: string[],
  exclude: string[],
): Promise<string> {
  const root = await fs.realpath(repoRootInput);
  const lexical = path.resolve(root, candidateInput);
  const existingParent = await nearestExistingParent(lexical);
  const realParent = await fs.realpath(existingParent);
  if (!isContained(root, realParent)) throw new Error("symlink/path escape");

  const relative = path.relative(root, lexical).split(path.sep).join("/");
  if (relative === "" || relative.startsWith("../")) throw new Error("outside repo");
  if (exclude.some((g) => picomatch.isMatch(relative, g))) throw new Error("excluded path");
  if (!include.some((g) => picomatch.isMatch(relative, g))) throw new Error("outside filescope");
  return relative;
}
```

### 写后复核

不要只相信 agent 自报改了哪些文件。用 Git 读取 tracked + untracked 实际集合，并逐一过 scope。使用 NUL 分隔输出，避免奇怪文件名破坏解析。

```text
git diff --name-only -z
git diff --cached --name-only -z
git ls-files --others --exclude-standard -z
```

发现越界时：

- 立刻 BLOCKED；
- 保存证据；
- 不自动删除用户文件；
- 不用 `git reset --hard` 自作主张恢复；
- 向 Owner 报告精确路径与安全处置选项。

---

## 13. ACTIVE-WORK 与冲突检测

唯一账本至少记录：

```markdown
| task_id | base | branch/worktree | filescope | conflict_set | status | rollback |
|---|---|---|---|---|---|---|
```

冲突不只包括同一文件：

- parent/child 路径；
- lockfile；
- database schema/migration；
- launcher 与 live config；
- generated artifact 与 source；
- shared prompt registry；
- 同一测试 fixture；
- 同一 Sheet row 或 Outbox 文件名。

默认并发：

```text
1 个写任务 + 1 个只读评审
```

任何不能证明无交集的 filescope 都串行。

Worktree 创建前登记，终态只有：

- removed；
- frozen + 理由 + 恢复命令；
- canonical/integrated。

“做完先留着”不是终态。

---

## 14. Secret、隐私与日志

### 三层防护

1. **权限隔离**：执行进程根本拿不到不需要的 secret；
2. **schema 限界**：packet 不允许 secret 字段；
3. **检测与脱敏**：作为最后一道补充。

不要把正则扫描当成秘密保险箱。

```ts
const SECRET_PATTERNS = [
  /-----BEGIN [A-Z ]*PRIVATE KEY-----/,
  /\b(?:access|refresh|api)[_-]?token\b\s*[:=]/i,
  /\bclient[_-]?secret\b\s*[:=]/i,
  /\bpassword\b\s*[:=]/i,
];

export function assertNoSecretLikeText(text: string): void {
  if (SECRET_PATTERNS.some((re) => re.test(text))) {
    throw new Error("secret-like content rejected");
  }
}

export function boundedError(error: unknown): string {
  const raw = error instanceof Error ? error.message : String(error);
  return raw
    .replace(/\b(token|password|secret)=\S+/gi, "$1=[REDACTED]")
    .slice(0, 500);
}
```

`boundedError` 仍必须配合单元测试和结构化日志字段；它只是避免常见明文泄露，不是通用 secret detector。

### 日志允许写什么？

```text
task_id, packet_hash, status, decision, rule_id,
tool name, bounded duration, exit code, artifact hash
```

默认不写：

```text
完整 body、原始模型输出、环境变量、argv secret、system prompt、transcript、私人记忆
```

---

## 15. KILL：所有入口共享的硬门

```ts
// kill.ts
import { access } from "node:fs/promises";
import { constants } from "node:fs";

export async function assertNotKilled(killPath: string): Promise<void> {
  try {
    await access(killPath, constants.F_OK);
    throw new Error("RELAY_KILLED");
  } catch (error: any) {
    if (error?.message === "RELAY_KILLED") throw error;
    if (error?.code !== "ENOENT") throw error;
  }
}
```

检查时点：

- scan 前；
- claim 前；
- executor 启动前；
- 每个高影响 tool call 前；
- commit/merge/deploy 前；
- Sheet 状态推进前；
- watcher 每轮。

KILL 存在时只允许：

- 读状态；
- 写一条不含私密正文的 KILL receipt；
- 安全释放本任务临时锁。

Agent 不得自行删除、重命名或绕过 KILL。

---

## 16. 安全调用 Claude Agent SDK

### 最重要的权限坑

官方权限语义中：

- `allowedTools` 只是预批准，不天然等于“只能用这些工具”；
- `bypassPermissions` 会放行未列出的工具；
- `acceptEdits` 也可能批准 mkdir/rm/mv 等文件操作；
- bare allow rule 会让对应 tool 绕过 `canUseTool`；
- 若必须检查每次调用，应使用 PreToolUse hook；
- `dontAsk` + 精确 allowlist 才是 headless hard-deny 模式之一。

因此不要写：

```ts
// DANGEROUS: allowedTools does not constrain bypassPermissions
options: {
  allowedTools: ["Read"],
  permissionMode: "bypassPermissions"
}
```

### 推荐策略

读工具可以 bare allow；Edit/Write/Bash 走可验证规则。最简单的 headless 模式是：

- `permissionMode: "default"`；
- 只 bare allow Read/Glob/Grep；
- Edit/Write/Bash 由 `canUseTool` 决定；
- 再加 PreToolUse hook 做不可绕过的 filescope/KILL 检查；
- 不使用 bypassPermissions；
- 设置 maxTurns、maxBudgetUsd 和 AbortController timeout；
- `settingSources` 明确指定，避免意外继承用户级宽权限。

```ts
// executor.ts — illustrative; adapt input keys to the pinned SDK version
import { query } from "@anthropic-ai/claude-agent-sdk";

export async function runBoundedClaudeTask(input: {
  cwd: string;
  prompt: string;
  allowedFiles: (path: string) => Promise<boolean>;
  exactBashCommands: Set<string>;
  timeoutMs: number;
}) {
  const abortController = new AbortController();
  const timer = setTimeout(() => abortController.abort(), input.timeoutMs);

  try {
    for await (const message of query({
      prompt: input.prompt,
      options: {
        cwd: input.cwd,
        settingSources: [],
        permissionMode: "default",
        allowedTools: ["Read", "Glob", "Grep"],
        disallowedTools: [
          "Bash(rm *)",
          "Bash(git push *)",
          "Bash(git reset --hard *)",
          "Bash(git clean *)",
        ],
        maxTurns: 30,
        maxBudgetUsd: 5,
        abortController,
        canUseTool: async (toolName, toolInput, options) => {
          if (options.signal.aborted) {
            return { behavior: "deny", message: "run aborted", interrupt: true };
          }

          if (toolName === "Edit" || toolName === "Write") {
            const p = String(toolInput.file_path ?? toolInput.path ?? "");
            if (!p || !(await input.allowedFiles(p))) {
              return { behavior: "deny", message: "path outside filescope", interrupt: true };
            }
            return { behavior: "allow", updatedInput: toolInput };
          }

          if (toolName === "Bash") {
            const command = String(toolInput.command ?? "");
            if (!input.exactBashCommands.has(command)) {
              return { behavior: "deny", message: "command not pre-authorized" };
            }
            return { behavior: "allow", updatedInput: toolInput };
          }

          return { behavior: "deny", message: `tool not allowed: ${toolName}` };
        },
      },
    })) {
      // Store bounded structured events. Do not dump raw private content to logs.
      if ("result" in message) return message.result;
    }
  } finally {
    clearTimeout(timer);
  }
}
```

### 为什么 Bash 用 exact set？

Shell 命令很难靠字符串前缀安全判断：

```text
npm test && curl ...
git status; destructive-command
echo $(nested-command)
```

Foreman 应从可信 repo policy 生成完整允许集合，例如：

```ts
new Set([
  "npm test",
  "npm run typecheck",
  "git status --short",
  "git diff --stat",
]);
```

更稳的方案是不用通用 Bash，给 executor 暴露 `run_tests`、`run_typecheck`、`git_diff` 等固定 wrapper/MCP tools。

### `settingSources` 的坑

默认 SDK 可能加载 user/project/local settings。自动化 wrapper 应明确选择：

- `[]`：最可预测，但需要显式提供必要项目说明；
- `["project"]`：加载项目规则，但必须先审计 `.claude/settings.json`；
- 不要无意加载 local/user 宽权限。

若需要 `CLAUDE.md`，可以在经过审计后使用 project source，或由 Foreman 读取可信项目指令并放入 bounded prompt。

---

## 17. PreToolUse hook：硬门要比模型更早运行

`canUseTool` 可能被早先的 allow rule 绕过；PreToolUse hook 适合执行每次都必须检查的 KILL 和 filescope。

概念配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/relay-hooks/pre-tool-guard.mjs",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Guard 必须：

- 从 stdin 解析 JSON；
- 验证 event schema；
- 重新检查 KILL；
- 读取当前 task 的不可变 scope snapshot；
- 对路径 realpath 检查；
- 对 Bash 只认精确授权；
- 返回结构化 allow/deny；
- 超时或异常时 deny（fail-closed）；
- 不输出 packet 私密正文。

Hook 以当前系统用户权限运行，不是 sandbox。脚本本身也要防路径穿越、变量未引用和敏感文件读取。

---

## 18. 人工批准：暂停、持久化、精确恢复

高影响动作出现时，在副作用前生成 approval card：

```ts
import { createHash, randomUUID } from "node:crypto";

type ApprovalAction = {
  task_id: string;
  action: string;
  parameters: Record<string, unknown>;
  impact: string;
  rollback: string;
};

function actionHash(action: ApprovalAction): string {
  return `sha256:${createHash("sha256")
    .update(canonicalJson(action), "utf8")
    .digest("hex")}`;
}

const approval = {
  approval_id: randomUUID(),
  nonce: randomUUID(),
  action_hash: actionHash(action),
  status: "pending",
  expires_at: new Date(Date.now() + 24 * 60 * 60_000).toISOString(),
};
```

Owner 的决定必须绑定：

```text
task_id + revision + approval_id + nonce + action_hash + expiry
```

任何 action 参数变化都会让旧批准失效。

允许三种决定：

- approve；
- reject；
- edit-and-resubmit（产生新 action hash，再确认）。

不要接受：

- 另一个聊天里孤立的“可以”；
- 没有 task ID 的确认；
- 过期 nonce；
- 对旧 revision 的批准；
- “批准部署”被拿去批准删除。

如果框架支持 interrupt/resume，持久化 thread/run ID；如果没有，`awaiting_owner` + approval record 就能实现同样语义。

---

## 19. Receipt 与 Outbox：避免自引用哈希

receipt 的哈希不能包含它自己的 hash 字段。

```ts
// receipt.ts
import { createHash } from "node:crypto";

export function signReceipt(unsigned: Record<string, unknown>) {
  if ("receipt_sha256" in unsigned) throw new Error("unsigned receipt contains hash");
  const digest = createHash("sha256")
    .update(canonicalJson(unsigned), "utf8")
    .digest("hex");
  return { ...unsigned, receipt_sha256: `sha256:${digest}` };
}
```

机器 receipt 至少包含：

```json
{
  "task_id": "DEMO-001",
  "revision": 0,
  "idempotency_key": "sha256:...",
  "status": "complete",
  "classification": "AUTO",
  "rule_id": "BOUNDED_AUTO_V1",
  "policy_version": 1,
  "base_before": "0123abc",
  "head_after": "4567def",
  "tests": ["unit: pass", "typecheck: pass"],
  "files_changed": ["src/status/render.ts"],
  "live_touched": false,
  "started_at": "...",
  "completed_at": "...",
  "receipt_sha256": "sha256:..."
}
```

人类报告至少包含：

- old → new；
- 修改文件；
- 测试与退出码；
- live 是否触碰；
- 未执行项；
- 风险与未决项；
- rollback；
- receipt hash。

上传步骤：

1. 本地写临时文件；
2. fsync/原子 rename 成最终文件；
3. 计算文件 SHA-256；
4. 使用不可覆盖语义上传；
5. 从远端下载到全新临时路径；
6. 对照 SHA-256；
7. Planner 按精确文件名读取；
8. 最后才把 Sheet row 推进 complete。

若第 4–7 步失败，代码可以完成，但 Relay 状态应为 `blocked` 或 `delivery_pending`，不能谎报端到端 complete。

---

## 20. Watcher：不要用重叠的 `setInterval`

错误写法：

```ts
// WRONG: a slow scan overlaps the next scan
setInterval(scanAndRun, 5_000);
```

正确方向是上一轮结束后再安排下一轮，并加入 jitter 与退避：

```ts
// watcher.ts
const sleep = (ms: number, signal: AbortSignal) =>
  new Promise<void>((resolve, reject) => {
    const timer = setTimeout(resolve, ms);
    signal.addEventListener("abort", () => {
      clearTimeout(timer);
      reject(new Error("aborted"));
    }, { once: true });
  });

export async function watch(signal: AbortSignal) {
  let failures = 0;
  while (!signal.aborted) {
    try {
      await assertNotKilled(process.env.RELAY_KILL_PATH!);
      await withProcessLock(process.env.RELAY_LOCK_ROOT!, "writer", async () => {
        const tasks = await listAuthorizedTasks({ limit: 1 });
        for (const task of tasks) await processOne(task);
      });
      failures = 0;
    } catch (error) {
      failures += 1;
      await writeBoundedWatcherError(error);
    }

    const base = failures === 0 ? 30_000 : Math.min(15 * 60_000, 30_000 * 2 ** failures);
    const jitter = Math.floor(Math.random() * 5_000);
    await sleep(base + jitter, signal);
  }
}
```

示例省略了函数实现；第一次施工的小机必须补齐并测试，不能复制后宣称 watcher 完成。

Watcher 还必须限制：

- 每批最多认领数；
- 写任务并发 1；
- 每任务 timeout；
- maxTurns；
- maxBudget；
- 每日总预算；
- 连续失败熔断；
- 无人响应时 approval expiry；
- 优雅停止与 lease 处理。

### watcher 何时才允许上线？

只有当以下均通过：

- 手动 scan 至少多轮无重复；
- crash/retry 演练；
- KILL 演练；
- approval 暂停/恢复演练；
- 断网与 API 限流演练；
- 费用上限演练；
- Owner 明确授权 daemon/watcher。

---

## 21. Git、worktree 与集成

施工前记录：

```text
repo realpath
current branch
HEAD
git status --porcelain=v1 -z
registered worktrees
live process version（若有关）
```

规则：

- 不在脏 canonical 上盲目施工；
- 不自动丢弃用户改动；
- branch/worktree 由 Foreman 统一命名与登记；
- 子 agent 不自建二层 worktree；
- rebase/merge 只有工单明确授权才做；
- 只允许 ff-only 集成时，失败就 STOP；
- force、reset hard、clean、branch delete 永远高风险；
- commit 前再次检查 diff 与 filescope；
- 集成后再次运行门禁，而非只信分支测试。

回滚锚应在副作用之前创建，并写入 receipt。不要把“git 里有历史”当成数据库、云权限或 live 状态的完整回滚。

---

## 22. 备份与不可逆动作

删除 worktree、ref、数据库、云文件或迁移前：

1. 明确精确目标；
2. 只读检查目标当前状态；
3. 证明备份覆盖目标；
4. 在隔离位置完成恢复演练；
5. 对照哈希；
6. 获得对应授权；
7. 执行后记录删除内容与恢复方法。

同一物理盘的两个目录不是 off-device redundancy。云端“上传成功”也不是恢复验证。

---

## 23. 测试策略：先 fake，再探针，再真实任务

### Unit tests

```text
[ ] strict schema rejects unknown fields
[ ] arrays parse as JSON, not comma split
[ ] formula-like cells rejected
[ ] RAW output preserved
[ ] canonical JSON stable across object key order
[ ] array order changes hash
[ ] runtime fields do not change logical task key
[ ] duplicate ID/key rules correct
[ ] every illegal state transition rejected
[ ] unknown action fails closed
[ ] high-impact action always ASK_OWNER
[ ] excluded path wins over include
[ ] /repo-evil does not pass /repo check
[ ] symlink escape rejected
[ ] secret fixture rejected and redacted
[ ] receipt hash excludes its own field
```

### Concurrency and recovery tests

```text
[ ] two local workers: only one lock succeeds
[ ] two claims: only one owning token survives
[ ] row reordered between read/write: identity check blocks
[ ] collaborator edits status: write-back blocks
[ ] crash before side effect: safe retry
[ ] crash after side effect before receipt: reconcile, no blind retry
[ ] expired lease does not auto-authorize re-execution
[ ] watcher restart does not duplicate task
```

### Permission tests

```text
[ ] Read/Glob/Grep work
[ ] Edit inside scope works
[ ] Edit outside scope denied
[ ] new file through symlink denied
[ ] unknown Bash denied
[ ] exact test command works
[ ] chained shell command denied
[ ] rm/git push/reset hard denied
[ ] KILL denies every write/tool path
[ ] SDK settingSources cannot import unexpected allow rules
[ ] subagent cannot inherit broader permissions than intended
```

### End-to-end fixtures

1. `authorized/AUTO` 只读任务完成；
2. `proposed` 不被消费；
3. `ASK_OWNER` 副作用前暂停；
4. reject 后 blocked；
5. 修改 action 后旧批准失效；
6. KILL 对 scan/claim/execute/advance/watcher 全拒；
7. 重复 task ID 拒绝；
8. 重复 idempotency key 拒绝；
9. stale base 阻塞；
10. filescope 冲突阻塞；
11. CSV 逗号、引号、换行往返；
12. 公式内容不执行；
13. claim 后 crash 不重复副作用；
14. receipt 只更新 owning-row；
15. Outbox 可被 Planner 精确读取并核对哈希；
16. timeout、maxTurns、budget 生效；
17. live/deploy 无批准时不可达。

### 真实通道探针

用随机 challenge，不放隐私：

```text
Planner 写 challenge
→ Foreman 读回并逐字匹配
→ Foreman 写 ACK + hash
→ Planner 读回并逐字匹配
```

四步全过才叫“双向闭环”。

---

## 24. 分阶段实施顺序

### Phase 0 — 设计与只读审计

交付：环境图、人工挡位、policy、schema、风险与 STOP 条件。零代码、零权限变更。

### Phase 1 — 单向 Planner → Inbox

Planner 能写探针；Foreman 只读展示，不认领。

### Phase 2 — 双向状态与 receipt

Foreman 可认领只读 fixture、写 receipt、Planner 回读。仍不碰项目代码。

### Phase 3 — ASK_OWNER 演练

高影响 fixture 暂停、批准、拒绝、过期、编辑后重新确认全部通过。

### Phase 4 — 有界代码施工

只在隔离 repo/worktree；scope、测试、diff、commit、回滚门全过。

### Phase 5 — 人工叫醒的正式 Relay

挡位 2 上线。观察重复、冲突、回执和人类负担。

### Phase 6 — watcher（单独授权）

新增 daemon/定时器、锁、lease、budget、熔断、KILL 与监控。

### Phase 7 — 自动 deploy（可选、单独授权）

仅 staging/健康检查/备份/回滚/required reviewer 均成熟后评估。不是 Relay 的必选终点。

---

## 25. 故障速查

| 现象 | 常见原因 | 安全处理 |
|---|---|---|
| Planner 看得到，Foreman 看不到 | `drive.file` 创建者可见性 | 让 Foreman/app 创建文件并重做双向探针 |
| Foreman 看得到，Planner 不能写 | ChatGPT App 无写 action/需批准 | 使用按钮、MCP、Apps Script 或保留人工投递 |
| 同一任务跑两次 | 无幂等、claim 或 crash reconcile | 停 watcher，核对 receipt/副作用，修复后再开 |
| 状态写错行 | 依赖缓存行号/标题匹配 | 按 task ID + key 重新定位，写后回读 |
| 一直重复询问 | policy 未持久化/规则无版本 | 保存 policy，receipt 记录 rule ID |
| agent 越出 scope | 字符串路径检查/权限过宽 | KILL，保留证据，修 realpath + PreToolUse guard |
| allowedTools 后仍能用别的工具 | 误解 SDK 权限语义 | 不用 bypass；dontAsk 或 default + callback/hook |
| Sheet 文字变成公式 | USER_ENTERED/网页输入 | FORMULA 读取检查，RAW 写入 |
| active 永远不结束 | worker crash/lease 无恢复 | reconcile 副作用，勿直接重新授权 |
| 上传了但 Planner 找不到 | 出腿成功≠connector 回读 | 精确文件名、权限和 challenge 回读 |
| worktree 越来越多 | 无一进一出账本纪律 | 停止新建，逐个审计、冻结或移除 |

---

## 26. 首次交付必须包含什么？

第一次施工的小机在宣布完成前必须交付：

```text
1. 架构图与实际组件清单
2. 人工挡位与 policy version
3. 文件与目录清单
4. schema 与 migration 说明
5. 权限/secret/KILL 边界
6. 所有测试及真实输出摘要
7. 双向 challenge 证据
8. watcher/live 是否明确未实现
9. 已知限制
10. rollback 与完整删除方法
11. 下一项需要 Owner 决定的事项
```

不要只交“tests passed”。必须让下一位人或机器能从文件和收据恢复现场。

---

## 27. 官方资料

- [Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in)
- [Claude Agent SDK TypeScript reference](https://code.claude.com/docs/en/agent-sdk/typescript)
- [Claude Agent SDK permissions](https://code.claude.com/docs/en/agent-sdk/permissions)
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks)
- [Google Sheets read/write values](https://developers.google.com/workspace/sheets/api/guides/values)
- [Google Sheets batch requests](https://developers.google.com/workspace/sheets/api/guides/batch)
- [Google Drive API scopes](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- [rclone Google Drive backend](https://rclone.org/drive/)
- [LangGraph interrupts](https://docs.langchain.com/oss/javascript/langgraph/interrupts)
- [GitHub deployment protection rules](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)

这些链接是能力与安全语义的来源；实现时仍需针对你锁定的版本重新核对参数与默认行为。

## Credits

Built by Gwendolen with Amelia GPT and Amelia Claude.
