# M0 实施步骤（单机闭环）

> 版本目录：v0.1.2 · 配套 `../prd.md` §11 · 日期：2026-09-09
> 本文档把 M0 拆到"照着就能做"的粒度。**技术选型是默认假设，开工前需拍板（主干 §14-11）。**

---

## 0. M0 目标与验收

**目标**：一台机器上跑 Navigator + 一个本地 worker（真实 WS 隧道连 localhost），跑通：

```
建项目(绑仓库) → 定故事卡目标 → 拆任务(手动兜底) → 派发 → agent 在 worktree 执行自验
→ commit/push → 目标级验收 → 通过则合并主分支 → Web 看 evidence
```

**验收**：手动跑通上述链路一次；本机跑的与将来远程**同一条代码路径**（worker 不因"在 localhost"走任何捷径）。

**M0 不含**（M1+）：Spark 板/汇总分类/澄清、大脑规划（目标拆解先手动）、语义校验、架构页、远程机器、Mattermost。

---

## 1. 技术选型（默认假设，开工前拍板）

| 项 | 默认假设 | 备选 |
|---|---|---|
| 语言/运行时 | **Node.js + TypeScript**（与 cumora/schedule-task 生态一致） | Python |
| 仓库结构 | monorepo：`server/`（Navigator）`worker/`（daemon）`shared/`（类型与协议） | 单包 |
| 状态库 | SQLite（better-sqlite3），Navigator 单写者 | Postgres |
| HTTP/WS | Fastify（HTTP）+ `ws`（WebSocket server/client） | Express |
| Web UI | 极简：静态页 + fetch（先不做框架，M1 再定） | React/Vue |
| 登录 | 用户名/密码 + bcrypt + 会话 token | — |
| ACP 驱动 | 子进程 stdio → JSON-RPC 桥接（见 §5 spike） | — |
| 合并 | v1 Navigator 本体确定性合并（`git merge`），假设无冲突 | 冲突留给 M1 大脑 |

---

## 2. 步骤（每步都有验证）

### S1 · Spike：agent CLI 的 ACP 支持（最先做，风险最高）

- 目标：确认本机要驱动的 agent（CC / Kimi / opencode）哪个支持 **ACP server 模式**，怎么启动、走 stdio 还是 TCP、握手格式。
- 动作：各装/查版本；`--help` 找 acp 相关 flag；最小起一个会话，发 `session/new` + `session/prompt` 看回包。
- **验证**：记录下"哪个 CLI + 启动命令 + 消息样例"，写进 `docs/design/v0.1.2/specs/connectivity-notes.md`（实测记录，非空想）。
- 若某个 CLI 不支持 ACP：准备降级路径（见 S6 风险）。

### S2 · 脚手架 + 数据模型

- monorepo 初始化（TS、lint、`shared/` 类型包）。
- SQLite schema（字段以主干 §13 为准，M0 只建用得到的表）：

```
projects(id,name) · repos(id,project_id,path,branch) · goals(id,project_id,story_json,state,worktree_path)
tasks(id,goal_id,title,requirement,state,worker_id) · workers(id,name,state,last_heartbeat,token_hash)
sessions(id,task_id,worker_id,state) · evidence(id,task_id,commit_ref,summary,created_at)
sparks(id,project_id,text,source,status,created_at)   -- 仅验收落卡用，板功能 M1
```

- **验证**：migration 可跑、建删项目/目标/任务 CRUD 通过（脚本测试）。

### S3 · HTTP API 最小集 + 登录

- `POST /api/login`（用户名/密码 → token）；中间件保护其余路由。
- 项目/目标/任务的基础 CRUD + 列表（先不做完整 UI，接口先行）。
- **验证**：curl 走通 登录→建项目→建目标→建任务；未登录 401。

### S4 · WS 连通层（配对 + 心跳）

- `server/`：WS server，连接登记为"未授权 worker"。
- `worker/`：daemon 以 WS client 外连 `ws://localhost:PORT`；启动生成**配对密钥**并显示在终端（tmux）。
- Navigator 侧批准：`POST /api/workers/:id/approve {pairing_key}` → 发**设备凭证**（随机 token，存 hash）；worker 之后重连带凭证自动认证。
- 心跳：worker 每 N 秒发 ping；Navigator 记录 `last_heartbeat`，超时标 offline。
- **验证**：起 server → 起 daemon → 看"未授权"→ 批准 → 变 online；杀 daemon → 心跳超时变 offline；凭证吊销后重连被拒。

### S5 · worker 收"开会话"并拉起 agent（ACP 桥接）

- `worker/` 收到 Navigator 的 `session/open {task_id, worktree, engine}` 消息：
  1. 确认 worktree 存在（没有则按 S7 建）；
  2. 在 worktree 目录**懒启动** agent 子进程（S1 确认的 ACP 模式）；
  3. 桥接：Navigator 发来的 JSON-RPC → 写子进程 stdin；子进程 stdout → 回 WS。
- 会话结束（agent 退出/超时）→ 上报关闭，回收。
- **验证**：Navigator 能经 WS 给本机 agent 发一句 prompt 并拿到回复（等价于"借到了本地脑力"）。

### S6 · 派发（能力 + 空闲即派，无婉拒）

- worker 启动上报能力（v1 简化：静态配置"能跑哪些 engine"）。
- `POST /api/goals/:id/tasks/:id/dispatch`：找 online+idle+有能力 的 worker → 推"开会话"→ 任务 `queued→running`（worker 回报开始）→ 完成回报 `done` + evidence。
- 任务状态迁移只由 worker 上报触发；心跳丢失 → 任务 `interrupted`，可重派。
- **风险/降级**：S1 若发现某 CLI 不支持 ACP——降级路径：daemon 用该 CLI 的非 ACP 模式（如纯 `-p prompt` 一次性执行）包一层，能跑通 M0 即可，ACP 会话留到 M1；在 connectivity-notes 记录。

### S7 · git worktree + commit/push + evidence

- `POST /api/projects/:id/repos` 绑定时，本地准备：v1 用**本地 bare 仓库**当"主分支"（`repo_path/bare.git` + 一个 worktree 目录），避免污染真实仓库，且天然可测 merge。
- 目标开工：`git worktree add <goal-worktree> -b goal/<id>`（从 bare 的 main 检出）。
- agent 干完：在 goal-worktree 里 commit + push 到 bare 的 `goal/<id>` 分支。
- 上报 evidence：`{commit_ref, summary}`（diff stat / 变更文件清单）。
- **验证**：agent 的改动出现在 bare 仓库的 goal 分支；evidence 记录可查。

### S8 · 目标级验收 + 合并

- Web/API：列出目标所有任务 + evidence → 人验收 `POST /api/goals/:id/accept {pass: true|false, note}`。
- **通过**：Navigator 本体在 bare 仓库执行 `git merge goal/<id> → main` → goal `done`。
- **不通过**：自动落一张 Spark（`source=acceptance, text=note`）→ goal 置 `cancelled`（N3 默认，见 goal-story-card spec §5）。
- **验证**：通过→main 有新提交；不通过→Spark 有记录、main 无该分支改动。

### S9 · 极简 Web 界面

- 登录页；项目列表；目标页（故事卡三要素 + 验收标准 + 边界 + 任务清单 + 每任务 evidence）；验收按钮。
- 样式不追求——**可见性第一**（参考 loopx dashboard 的信息组织，不做它的 UI）。
- **验证**：浏览器全程走完 M0 链路。

### S10 · 端到端验收 + 文档收尾

- 按 §0 验收清单手动跑一遍；把踩坑（尤其 S1 的 ACP 实测）回填 `connectivity-notes.md`。
- 确定哪些 M1 项要提前（视 S1 spike 结果）。

---

## 3. 风险与缓解

| 风险 | 缓解 |
|---|---|
| agent CLI ACP 支持不成熟（最高风险） | S1 最先做；不行就降级非 ACP 一次性执行包一层 |
| 合并冲突 | v1 Navigator 确定性合并，假设无冲突；冲突出现 → 人手工解决 + M1 再议大脑合并 |
| SQLite 单文件并发写 | 单写者设计：所有写走 server 进程 |
| "单机捷径"渗透进代码 | 评审标准：worker 代码不允许"if host==localhost 就直连"，一律走 WS |

---

## 4. 完成定义（Definition of Done）

- S1–S10 全部验证通过；
- 工作区代码路径与远程设计一致（无单机特例）；
- 主干 §14 待决策中与 M0 相关的项（11 技术选型、1 合并执行者）已拍板或记录临时默认。
