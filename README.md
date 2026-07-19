# Relay Between Agents

**让一个 AI 负责规划，另一个 AI 负责施工，而人类不再夹在中间反复复制粘贴。**

Relay Between Agents 是一套轻量、可审计、可逐步自动化的多 Agent 协作方法。它把“讨论需求”和“真正动手”拆成两个角色，再用一张任务表、明确的授权规则和可回读的施工报告把它们连接起来。

你不需要先懂 agent framework，也不需要把项目改造成复杂平台。最小版本只要：

```text
规划 AI 写下任务
→ 施工 AI 读取任务
→ 按授权等级执行或停下来询问
→ 把结果与证据写回
→ 规划 AI 直接读取并验收
```

人类仍然拥有最终控制权，但不再充当两个 AI 之间的剪贴板。

---

## 先看这里：这份教程适合谁？

只要你的项目同时存在“需要想清楚”和“需要真正执行”两类工作，就可以使用 Relay。它不限于编程，也不限于个人 AI 或人机关系项目。

| 场景 | 规划 AI 可以做什么 | 施工 AI 可以做什么 |
|---|---|---|
| 软件项目 | 梳理需求、排优先级、写验收标准 | 修改代码、运行测试、提交 PR |
| 研究项目 | 拆研究问题、规定来源与证据标准 | 搜集资料、清洗数据、生成报告 |
| 内容项目 | 决定主题、结构、语气和审核标准 | 写稿、排版、制作素材、发布草稿 |
| 个人知识库 | 判断什么值得记录、如何分类 | 整理文件、去重、建立索引 |
| 家庭自动化 | 制定规则、识别例外与风险 | 执行脚本、更新设备配置 |
| AI companion | 讨论陪伴体验和产品边界 | 实现前端、记忆、消息或多模态功能 |

特别适合以下情况：

- 你常在一个 AI 里把需求聊得很清楚，却要手动复制给另一个 coding agent；
- 施工报告很长，你没有精力反复搬运；
- 项目跨越很多天，聊天窗口容易忘记之前做到了哪里；
- 你希望低风险任务自动执行，但部署、删除、权限和隐私操作仍由自己确认；
- 你需要看得见每项任务为何被执行、为何暂停、改了什么、如何恢复。

它不太适合：

- 只有一次、两分钟就能做完的小任务；
- 没有任何外部写入能力、也不愿保留一份任务文件的纯聊天环境；
- 无法容忍机器犯错，却又准备跳过测试、备份和人工门禁的高风险系统；
- 希望两个 AI 无约束地互相聊天并自行扩大目标——Relay 刻意不这样设计。

> Relay 不是“让两个 AI 自由对话”。它是一条有任务编号、范围、状态、授权和收据的施工通道。

---

## 本教程基于什么环境？

这份教程首先讲我们已经实际走通的一套参考环境：

- **规划端：ChatGPT 官方端**  
  用来讨论需求、形成工单、读取交付报告并验收。
- **施工端：Claude Code 官方 CLI（下文简称 CC）**  
  在本地项目里读取文件、写代码、运行测试和维护施工账本。
- **中间通道：Google Sheet + Google Drive**  
  Sheet 作为 Inbox，Drive 文件夹作为 Outbox。
- **本地桥接：rclone + 小型验证脚本**  
  CC 读取 Sheet、验证任务、更新状态并上传报告。
- **默认运行方式：人工叫醒，自动施工**  
  人类只需说一句“查收 Relay”；不需要搬运工单和报告。

OpenAI 现在把过去的 connectors 统一称为 **Apps**。App 是否支持写入、是否每次需要批准，会受到套餐、工作区设置和具体 action 的影响。因此，第一步永远是用一条无隐私的探针任务实测“能否读、能否写”，不要只凭界面上出现了 Google Drive 就假设双向通道已经成立。

这不是唯一组合。教程后半部分会说明如何替换成：

- ChatGPT + Codex；
- Claude / Gemini / 其他官方前端 + 任意 coding agent；
- 自建网页前端 + OpenAI、Anthropic 或其他 API；
- GitHub Issues、数据库、消息队列或 MCP，而不是 Google Sheet；
- 一个模型的两个独立角色，而不是两家模型。

Relay 的核心不是品牌，而是下面四件事：

1. 规划与施工职责分开；
2. 工单是结构化且有界的；
3. 高影响操作能暂停等待人类；
4. 结果可以被另一端直接读取和审计。

---

## 90 秒理解 Relay

### 没有 Relay 时

```text
你和规划 AI 讨论需求
→ 复制几千字 prompt
→ 粘贴到 coding agent
→ 等施工
→ 复制几千字报告
→ 粘贴回规划 AI
→ 再复制修正意见
```

这条链的问题不只是麻烦。人类还是唯一的“状态同步器”：只要累了、忘了或贴错窗口，任务就可能丢失、重复或串线。

### 有 Relay 后

```text
Owner（人类）
   ↕ 讨论目标与授权
Planner（规划 AI）
   ↓ 写 authorized task
Inbox（任务表）
   ↓ 读取、校验、认领
Foreman（施工协调者 / CC）
   ↓ 写码、测试、记录
Outbox（报告与收据）
   ↑ 规划 AI 直接读取并审计
```

三个角色：

- **Owner**：机器、账号和项目的真正拥有者；决定放权到哪一档。
- **Planner**：理解目标、整理 backlog、写工单和做验收的 AI。
- **Foreman**：唯一施工协调者；刷新现场、认领工单、调用工具、集成修改并写收据。

Foreman 可以自己写代码，也可以调度子 agent。但对外只能有一个 Foreman 维护唯一施工账本，否则很快又会回到 worktree、分支和任务互相不知道的混乱状态。

---

## 先选人工挡位：你想亲自确认多少？

Relay 不要求你一步跳到“全自动”。最稳妥的方式是先选一个挡位，跑顺后再逐级放权。

| 挡位 | 人类需要做什么 | 自动做什么 | 适合谁 |
|---|---|---|---|
| 0. 手动搬运 | 复制工单，也复制报告 | 几乎没有 | 先体验角色分工 |
| 1. 每单确认 | 在表里批准每张工单，再叫醒施工端 | 搬运、校验、施工、回执 | 想看住每个任务 |
| 2. 委托施工（推荐） | 只叫醒一次；高影响动作再确认一次 | 有界代码、文档、测试、commit、报告 | 大多数个人项目 |
| 3. 自动接单 | 只处理高影响确认和异常 | watcher 自动发现任务并启动施工 | 通道已稳定的长期项目 |
| 4. 高度自动 | 查看摘要、处理异常、保管 KILL | 包括预先授权的部署 | 有 staging、监控和成熟回滚的项目 |

### 挡位 0：先不自动化

Planner 生成一张格式固定的工单，你仍手动贴给施工端；施工端按固定格式返回报告。虽然还在复制，但你可以先验证 schema、授权规则和报告是否真的好用。

### 挡位 1：每张工单都由你按下开始

Planner 把任务写成 `proposed`，你查看后把它改成 `authorized`，再说一句“查收 Relay”。Foreman 只认领 `authorized` 行。

适合刚搭好通道、尚未信任分类器时使用。

### 挡位 2：委托低风险施工（推荐默认）

Planner 可以直接写入 `authorized`。Foreman 自动完成：

- 只读检查、审计和测试；
- 工单 filescope 内的代码与文档修改；
- 工单明写的 commit、回滚锚和 ff-only 集成；
- 施工账本、状态和报告更新。

但以下操作仍暂停，只问你一次：

- live 部署、重启线上服务；
- 删除数据、删分支、force push、历史改写；
- OAuth、云端权限、密码、付费；
- 新 daemon、端口、webhook 或常驻 watcher；
- 以你的身份向外发布内容；
- 私密记忆、身份或关系 prompt；
- 任何超出工单 filescope 的扩张。

一次确认应覆盖它点名的完整动作。不要在“写完代码”“准备测试”“准备 commit”三个阶段重复询问同一件已经授权的事。

### 挡位 3：自动接单，但高风险仍等人

新增 watcher 或定时任务，定期查找 `authorized` 行；发现任务后自动运行 Foreman。遇到 `ASK_OWNER` 时保存状态并暂停，等你的批准再恢复。

这一档消除了“查收 Relay”这句门铃，但也新增了常驻进程、锁、重试、费用与权限风险。请先把挡位 2 连续跑稳，再单独实施。

### 挡位 4：高度自动，不等于没有边界

可以把经过 staging、测试、健康检查和回滚保护的部署也列入 AUTO。但密码、权限扩张、不可逆删除、外部身份行为等红线仍建议保留人工确认。

所谓“全自动”应该是：**在预先写清楚的授权范围内无需临场确认**，而不是让 agent 自己决定扩大授权范围。

---

## 从零搭建：人类可读版

下面先搭挡位 2：人工叫醒、低风险自动施工。它最能减少搬运，又不会一开始就引入后台 daemon。

### 第 1 步：先写一页边界，不要先写代码

在任何自动化之前，回答四个问题：

1. 哪个 AI 是 Planner？哪个进程是 Foreman？
2. 哪些项目或文件允许它们操作？
3. 什么可以自动，什么必须问人？
4. 出错时，哪个开关能让所有入口立刻停止？

可以直接采用这份初始政策：

```text
AUTO
- 只读检查、测试、typecheck、build
- 工单范围内的文档和代码修改
- 明确写入工单的 commit 与非破坏性集成
- 任务表、施工账本、收据和报告

ASK_OWNER
- live 部署与线上重启
- 删除、force、历史改写、不可逆迁移
- 密码、OAuth、Drive 权限与付费
- 新常驻服务、端口、webhook、watcher
- 以 Owner 身份对外发布
- 私密 prompt、记忆和 transcript
- 超出 filescope 的任何动作

BLOCKED
- KILL 开启
- 项目基线漂移
- 与正在施工的任务冲突
- 测试、验证或恢复门失败
- 工单本身自相矛盾
```

不要只把这段政策留在聊天里。将它保存成 Foreman 每次接单前必读的持久文件。

### 第 2 步：建立 Inbox 和 Outbox

参考实现使用：

```text
Google Drive/
└── Relay/
    ├── Relay-Inbox      # 一张原生 Google Sheet
    └── Relay-Outbox/    # 每项任务的人类可读报告
```

Inbox 是任务运输层，不是产品 backlog，也不是施工账本。三者职责不同：

- **Backlog**：未来想做什么、为什么、优先级如何；
- **Inbox**：这一次授权交给 Foreman 的具体任务；
- **施工账本**：现在谁正在做什么、在哪个基线、会改哪些文件。

把它们混成一张表，后面很容易出现“讨论过 = 已授权”“上传了 = 已执行”的误判。

### 第 3 步：连接 ChatGPT 官方端

在 ChatGPT 中打开 Apps/插件目录，连接 Google Drive。不同套餐、工作区和地区的可用能力可能不同，因此按实际界面核对：

1. Planner 能否找到指定 Sheet；
2. Planner 能否读取某一行；
3. Planner 能否更新一条无隐私探针；
4. 写入动作是否每次要求 ChatGPT 侧确认；
5. Planner 能否按精确文件名读取 Outbox 报告。

如果只能读不能写，Relay 仍可用：让 Planner 生成结构化工单，由一个最小 MCP、Google Apps Script、GitHub Issue 或人工按钮负责写入 Inbox。不要为了省一步而直接授予整个 Drive 的广泛写权限。

### 第 4 步：连接施工端

在参考实现中，CC 在本地工作，因此需要一种读取 Sheet、写回状态和上传报告的方法。rclone 是一个可选的轻量桥接方式。

若使用 Google Drive，优先评估 `drive.file`：它把 app 的访问范围限制在由该 app 创建或明确共享给它的文件。代价是网页手工创建的文件可能对 rclone 不可见。

所以必须做两条探针：

1. rclone 创建 Sheet，ChatGPT 能否读取和更新？
2. ChatGPT 或网页创建文件，rclone 能否看到？

我们的参考环境第一条成立，第二条受 `drive.file` 可见性限制。这个不对称性并非故障，而是最小权限的结果。

长期使用时：

- 不要把 rclone token、OAuth client secret 或配置文件提交到 GitHub；
- 若使用共享 client ID，记录它的退役或更换计划；
- 只给 Relay 专用目录和动作所需权限；
- 第一次双向连通必须用 challenge 或哈希回读验证。

### 第 5 步：设计一张人和机器都看得懂的任务表

最小列可以是：

| 列 | 人类理解 |
|---|---|
| `task_id` | 任务唯一编号 |
| `status` | 当前是待授权、施工中、完成还是阻塞 |
| `title` | 一眼能懂的任务名 |
| `body` | 完整施工说明 |
| `approval_level` | 自动执行还是需要询问 |
| `filescope` | 允许碰哪些文件或系统 |
| `excluded` | 明确不能碰什么 |
| `gates` | 必须通过哪些测试或检查 |
| `stop_conditions` | 发生什么必须立即停 |
| `base_expect` | 预期从哪个项目版本开始 |
| `idempotency_key` | 防止同一任务执行两遍的指纹 |
| `updated_by` | Planner、Foreman 还是 Owner 更新的 |
| `receipt_hash` | 最终报告的校验指纹 |
| `error` | 若阻塞，用一句话说明原因 |

第一版可以少一些列，但 `task_id`、`status`、`body`、授权等级、范围和停止条件不要省。

### 第 6 步：让 Planner 学会写好工单

一张可施工工单应回答：

- 要改变什么可观察行为？
- 为什么要做？完成后人类会看到什么？
- 哪些文件或系统能动，哪些绝对不能动？
- 必须通过哪些测试？
- 哪些情况出现时必须停止？
- 是否允许 commit、合并或部署？
- 报告需要包含哪些证据和回滚方式？

示例：

```yaml
task_id: DEMO-001
status: authorized
approval_level: auto
title: Add a bounded status field
base_expect: 0123abc
filescope:
  - src/status/**
  - tests/status/**
excluded:
  - credentials
  - identity prompts
  - database migrations
gates:
  - targeted tests
  - full typecheck
stop_conditions:
  - base drift
  - filescope conflict
  - permission expansion required
body: |
  在现有 status 输出末尾增加一个有界字段。
  其他输出保持不变，并补回归测试。
```

不要把密码、token、私人 prompt 或完整 transcript 放进工单。工单只应携带必要摘要、指针和哈希。

### 第 7 步：让 Foreman 按固定顺序接单

第一次接手这个项目的机器也应该能按照下面的顺序工作：

1. 检查 KILL 是否存在；
2. 读取授权政策和唯一施工账本；
3. 刷新仓库 HEAD、worktree、运行进程和脏文件；
4. 只寻找 `authorized` 工单；
5. 验证 schema、重复任务、基线和 filescope 冲突；
6. 重新分类为 `AUTO / ASK_OWNER / BLOCKED`，不能盲信表格自称的 auto；
7. 认领自己的那一行，推进为 `active`；
8. 在限定范围内施工和测试；
9. 写机器收据与人类报告；
10. 只把自己的那一行推进为 `complete` 或 `blocked`。

表格里写着 `auto` 只是 Planner 的请求，不是最终权限。Foreman 必须用本机持久政策重新判断。

### 第 8 步：报告不是“完成了”三个字

每项任务至少交付：

- 最终状态；
- 修改前 → 修改后；
- 精确修改文件；
- 测试和构建结果；
- 是否触碰 live；
- 没有做的事情；
- 风险与未决项；
- 回滚锚和恢复方法；
- 报告自身哈希。

把报告上传到 Outbox 后，还要让 Planner 从另一端按精确文件名读回来。`uploaded` 不等于 `connector-read-verified`。

### 第 9 步：做三次演练，再碰真实任务

至少演练：

1. 一个只读 AUTO 任务成功完成；
2. 一个 ASK_OWNER 任务停下来，确认前零执行；
3. KILL 存在时，所有入口都拒绝施工。

随后再测试重复 task ID、错误基线、冲突 filescope、崩溃后重试和报告回读。完整测试清单见 [ARCHITECTURE.md](./ARCHITECTURE.md)。

---

## 如果你希望每件事都手动确认

最简单的方法不是让 agent 每一步都弹窗，而是在状态机里设置清楚的暂停点。

推荐流程：

```text
Planner 写 proposed
→ Owner 将整张工单改为 authorized
→ Foreman 施工到新的高影响动作
→ 状态改为 awaiting_owner，并写清“准备做什么”
→ Owner approve / reject / edit
→ Foreman 从原状态恢复
```

确认卡应只包含：

- 即将执行的动作；
- 为什么需要它；
- 影响范围；
- 最坏后果；
- 回滚方法；
- `批准 / 修改后批准 / 拒绝` 三个选项。

不要问“可以继续吗？”这种没有信息量的问题，也不要在同一授权范围内反复确认。

如果你的执行框架支持 human-in-the-loop interrupt，可以直接把任务状态持久化后暂停；人类批准时用同一个 task ID 恢复。若没有框架，Sheet 的 `awaiting_owner` 状态本身就是一个简单、可见的 interrupt。

---

## 如果你希望自动接单和施工

“自动接单”比“自动写代码”多一个关键问题：**谁来叫醒施工 agent？**

有三种常见答案：

### 方案 A：定时轮询

每隔几分钟扫描一次 Inbox。实现简单、容易停用，适合个人项目。

必须具备：

- 单实例锁，避免两个 Foreman 同时认领；
- lease/超时，避免崩溃后任务永远卡在 active；
- 幂等键，避免重启后施工两次；
- 并发上限；
- 费用、运行时长和最大轮数限制；
- 失败退避，而不是每秒疯狂重试。

### 方案 B：事件或 webhook

任务表更新时立即通知 Foreman。响应更快，但新增公网端点、签名验证、重放防护和网络攻击面。除非项目本来就有安全的事件基础设施，否则不要为了省几分钟轮询而过早上 webhook。

### 方案 C：Agent SDK / 非交互 CLI

Watcher 发现任务后，通过 coding agent 的非交互模式或 SDK 启动一次有界施工。以 Claude Code 为例，官方 Agent SDK 支持设置 allowed tools、hooks、permissions 和 sessions；CLI/SDK 自动化时还应显式限制最大轮数、超时和可用工具。

无论使用哪种方法，全自动模式仍应保留：

- KILL；
- 本机授权分类器；
- 高风险暂停点；
- 测试和回滚；
- 每日或每批摘要；
- 无人回应时的安全超时；
- 不得自行扩 scope 的硬规则。

推荐的自动化上限是：低风险任务自动认领、自动施工、自动报告；部署和不可逆操作暂停等待人类。只有当 staging、监控、备份和回滚都经过真实演练后，再考虑自动部署。

---

## 如果不是 ChatGPT 官方端 + Claude Code，怎么替换？

把 Relay 看成四个可替换插槽：

```text
Planner Adapter
    ↓
Task Store / Inbox
    ↓
Executor Adapter
    ↓
Receipt Store / Outbox
```

### 自建前端 + API

你的前端可以提供三个按钮：

- 保存草稿；
- 授权施工；
- 批准 / 拒绝高影响动作。

后端只需要暴露类似接口：

```text
createTask()
listAuthorizedTasks()
claimTask()
requestApproval()
completeTask()
blockTask()
```

Planner 使用 OpenAI Responses API、Anthropic Messages API 或其他模型均可。Executor 可以使用 Claude Agent SDK、Codex、其他 coding agent，甚至固定脚本。模型不应该直接拿数据库写权限；它只能调用经过 schema 验证的工具。

### 没有 Google Drive

可以替换为：

- 私有 GitHub Issues / Projects；
- SQLite 或 Postgres；
- S3 兼容对象存储；
- Notion / Airtable；
- 消息队列；
- 一个只暴露 Relay 工具的 MCP server；
- 本地 `inbox/` 与 `outbox/` 文件夹。

传输层可以换，但以下语义不要丢：唯一 task ID、幂等、认领、状态机、审批、收据和 KILL。

### 两边都是同一个模型

也可以。让两个独立会话或进程承担不同职责：Planner 不持有施工工具，Foreman 不自行修改产品目标。角色隔离比模型品牌更重要。

### Planner 只能读，不能写

让 Planner 产出结构化 packet，由一个小型 MCP、按钮、Apps Script 或 GitHub Action 代写。最差情况下，人类只需按一次“投递”，仍比复制整份 prompt 和报告省力。

### Executor 不是 coding agent

只要它能执行有界动作并返回证据，就能成为 Foreman。例如研究 agent、排版 agent、数据处理脚本或家庭自动化控制器。

---

## 最容易踩的坑

### 1. 把聊天里的“好”当成授权

讨论、backlog 和 authorized task 是三种不同状态。只有明确进入授权状态的工单才能施工。

### 2. 表里写 auto，机器就真的全放行

Planner 可能判断错误或被恶意内容诱导。最终分类必须由施工机上的持久政策完成。

### 3. 上传成功就宣布双向闭环

另一端必须真的读取、核对 challenge 或哈希。上传成功只证明出腿，不证明回读和写回。

### 4. 最小 Drive 权限造成“看不见”

`drive.file` 下，app 往往只能看到自己创建或被明确共享的文件。先做两个方向的探针，再决定谁创建 Sheet。

### 5. 同一任务执行两次

网络重试、会话重启和后台轮询都会造成重复。必须使用 task ID、幂等键、owning-row 和 receipt 去重。

### 6. 每个任务都建 worktree，却没有收口

创建前登记，完成后删除或书面冻结；限制并发；Foreman 是唯一集成者。

### 7. Sheet 内容直接进入 shell

任何单元格都可能是恶意输入。使用真正的 CSV/JSON parser；不 `eval`，不执行公式，不把单元格拼成 shell 命令。

### 8. 把 secret 和私人正文写进 Outbox

报告只放必要摘要、哈希和证据指针。不要复制 token、环境变量、system prompt、私人记忆或 raw transcript。

### 9. 一上来就装 watcher

先让手动 scan 跑稳。否则只是把偶尔的人肉失误升级成常驻进程自动犯错。

### 10. 只有聊天记忆，没有账本

会话会结束、压缩或切换模型。任务状态必须存在于表、账本和收据里，而不是只存在于某个窗口的记忆里。

技术实现中最重要的状态一致性、安全解析和 crash recovery 细节见 [ARCHITECTURE.md](./ARCHITECTURE.md)。

---

## 可直接给第一次接手的小机的实施指令

<details>
<summary><strong>展开完整 Prompt</strong></summary>

```text
请为当前项目设计并实现一个安全、可审计的 Agent Relay。

先读本仓库 README.md 与 ARCHITECTURE.md。不要假设你记得先前聊天。

目标：
Planner 将结构化 authorized task 写入 Inbox；Foreman 读取后先验证 schema、KILL、基线、filescope、冲突与本机授权政策，再执行或暂停；状态只回写任务自己的 owning-row；完整报告进入 Outbox，供 Planner 直接读取和验收。

实施前先向 Owner 交付一页设计确认，必须写清：
1. 当前 Planner、Foreman、Inbox、Outbox 分别是什么；
2. 选择的人工挡位；
3. AUTO / ASK_OWNER / BLOCKED 边界；
4. 是否新增 daemon、端口、webhook、OAuth 或付费；
5. 秘密如何隔离；
6. KILL 与恢复路径；
7. 哪些能力需要先做双向探针。

默认实现挡位 2：人工叫醒、低风险自动施工。未经单独授权，不实现 watcher、自动部署或权限扩张。

最低实现要求：
- 唯一 task ID 与幂等键；
- authorized → active → complete|blocked 状态机；
- 持久授权政策，表中 auto 不能覆盖本机分类器；
- owning-row 精确更新；
- base drift 与 filescope 冲突检测；
- secret / 私密内容拒收；
- KILL 对所有入口生效；
- crash/retry 不重复施工；
- 机器 receipt + 人类可读报告；
- Outbox 回读验证；
- 一页操作手册和完整回滚方法。

先在隔离 fixture 上演练：
1. AUTO 只读任务成功；
2. ASK_OWNER 确认前零执行；
3. KILL 全拒；
4. 重复任务拒绝；
5. stale base 阻塞；
6. filescope 冲突阻塞；
7. 恶意 CSV/公式样式内容保持惰性；
8. 崩溃重试幂等；
9. receipt 只写 owning-row；
10. Planner 能从另一端读取并核对报告。

STOP 条件：任何需要扩大权限、触碰 live、创建常驻服务、执行破坏性操作、越出 filescope 或无法证明可恢复的情况。遇到 STOP 时只报告阻塞，不自行降低门禁。
```

</details>

---

## 官方资料与进一步阅读

- [Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in)
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview)
- [Google Drive API scopes](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- [Google Sheets values.update](https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/update)
- [rclone Google Drive backend](https://rclone.org/drive/)
- [LangGraph human-in-the-loop interrupts](https://docs.langchain.com/oss/javascript/langgraph/interrupts)
- [GitHub deployment protection and required reviewers](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)

这些资料共同验证了 Relay 采用的几个关键模式：外部 app 的读写与批准要分开管理；coding agent 的工具调用需要权限和生命周期门禁；人类确认应当以可恢复的暂停实现；部署可以通过独立保护规则等待 reviewer。

## License

[CC BY-NC-SA 4.0](./LICENSE.md)

## Credits

Relay Between Agents 来自一套真实的人类—AI 协作施工流程。公开教程中的示例经过重构与隐私清洗，只保留可复用的架构、权衡和踩坑经验。
