---
status: accepted
date: 2026-09-10
---

# 0018 · pg-core 协议与事件流（JSON-RPC 2.0 over stdio）

pg-core（Java Core）重建图（wayfinder #114）票 #119 的决策产物（grilling 2026-09-10，三轮收口；Round 1/2 主体按推荐，两处用户修正——消息分帧弃单行 JSON 改 Content-Length 长度帧、TS 生成源限定为协议契约层；Round 3 幂等与命名按推荐）。输入：[ADR-0016](0016-pg-core-process-model.md) 进程模型、[ADR-0017](0017-pg-core-tech-baseline.md) 技术基线、研究票 #117（lsp4j 事实）、旧栈 Go 契约（母仓 `docs/contract/04-p1-event-protocol.md`）。裁决依据：图定裁决尺（更容易理解 > 更容易调试 > 更容易重构 > 更少运行时魔法 > 更少隐式依赖 > 更少基础设施 > 更少需要记忆的规则），服务单人长期维护。

## 0. 决议清单

**线协议形态**

- **JSON-RPC 2.0 over stdio**：客户端 spawn 的 Core 子进程的 stdin/stdout 即协议通道；不占端口、不做服务发现，EOF 即断连感知。gRPC / HTTP+SSE / WebSocket 落选（基础设施重，或与无守护进程模型冲突）。
- **自写薄层，不引 lsp4j**（其 jsonrpc 模块传输层依赖 Gson，会与 Jackson 3.1 并存；ADR-0017 §1 备注随之作废）。自写范围限定七件：Request/Response/Notification、消息分帧、请求编号、等待应答登记（pending request）、分发器（dispatcher）、超时、取消。不得发展为通用 RPC 框架。
- **消息分帧**：LSP 式长度帧 `Content-Length: <bytes>\r\n\r\n<json>`，正文为 UTF-8 编码的 JSON，字节数按 UTF-8 计。不采用单行 JSON——协议不依赖「JSON 必须单行」，Pretty Print / 格式变化不构成隐式协议规则。
- **管道分工**：stdin/stdout 走协议，stderr 走日志与诊断；Core 业务代码禁止 `System.out.println`（防 stdout 污染）。EOF 触发有序关闭：停止接收新请求 → 停 watcher → 收口在途任务（恢复语义按 ADR-0016 靠下次启动）→ 关数据库 → 自然退出；不直接 `System.exit`，无宽限窗口。

**单 Core 多工作区前提**（用户补充的架构约束）

- 一个 Core 实例管理多个 Workspace（不是一 Core 一工作区）；协议、事件流、任务调度、数据库访问、状态缓存均以「单 Core、多 Workspace」为前提设计。跨客户端共享 daemon 仍不设（ADR-0016 口径不变：每客户端 spawn 自己的实例，该实例承担该客户端全部工作区）。
- Workspace 是一等协议实体，不是 Project 的附属字段。

**握手与版本**

- `initialize` 必须是第一条请求；未握手调用其他方法一律报错。
- params 携带 `client_protocol_version`（整数）。返回 `protocol_version`（整数，首版 1）、`core_version`（产品版本号）、`capabilities`（仅两份清单——方法集合与事件集合，不承担 DTO/schema 描述职责）。
- 版本策略：当前阶段严格相等，不等即拒并在错误中附双方版本；严格相等是现阶段策略，不固化为永久兼容规则。升号规则：加方法、加可选字段不升号；改语义、删改字段才升号。

**工作区（协议面）**

- 所有工作区域方法显式携带 `workspace_id`（第一参数）；不引入 workspace 句柄，不把路由编码进方法名。
- 显式生命周期：`workspace/open` 装载数据库、启动 watcher 等运行时资源；`workspace/close` 停止 watcher、释放资源。closed 状态拒收依赖该工作区的业务请求。有运行中 task 时 close 返回 `err.workspace.busy`；本轮不定义 force-close。
- 重复调用幂等：已 open 再 open 成功返回；未 open 而close 成功返回；不存在的 `workspace_id` 报 `err.workspace.not_found`。
- `workspace/list` 为发现入口（旧栈 `ListWorkspaces` 对应物）。工作区创建/删除/注册归域与存储票（#120/#121）。

**长操作与任务**

- 长操作统一立即返回 `task_id`；进度与终态经事件流 `task_updated` 推送；任务详情走查询接口。管道上只有短请求，长活在域内跑。
- 协议层 `$/cancelRequest` 仅作通用请求取消兜底；域级取消走任务自身的取消接口。CLI 在客户端侧等待任务终态并映射为命令退出码。

**事件流**

- 单一 live notification 流：Core 把它管理的全部工作区的事件推到同一管道；不维护订阅状态、不重放、不是事件总线。客户端不得依赖事件流恢复完整状态——事件丢失、断线、重启一律经重新查询恢复。
- 信封（旧栈 `EventEnvelope` 后继）定稿：`{schema_version, stream_epoch, stream_sequence, event_id, event_type, workspace_id, emitted_at, relation_id, task_id, payload}`。`workspace_id` 必带；`relation_id`/`task_id` 视事件类型可空。
- `schema_version`：事件协议自己的独立版本字段。旧栈查询面 DTO 上的 `schema_version` 全部砍除（前端从未在查询应答里消费过）。
- `stream_epoch`：Core 本次启动的唯一标识；`stream_sequence`：本次运行内从 1 递增，事件专用全局单调序号，不复用 commit/revision/snapshot 等业务版本号。客户端见 epoch 变化即本地事件相关状态全部失效并重查。跨重启无需持久化序号（#121 存储约束清零）。
- 客户端按 `workspace_id` + `event_type` + 事件语义决定是否重查；序号跳号只表示流中存在其他事件（其他工作区、其他客户端），不直接等价于当前工作区状态失效。
- 事件名延续 snake_case：`task_updated`、`relation_invalidated`、`watch_failed`。legacy 第二 topic `packgradle:mods-diff` 随 legacy 退场，新协议只有一条事件流；toast 触发若需要，走同一事件流表达。

**错误语义**

- `err.<域>.<原因>` 点分字符串码延续，码即客户端翻译键；结构 `{code, args, detail}`（code=机器语义+翻译键，args=文案变量，detail=技术诊断）。Core 不产生用户文案；CLI/Desktop/VS Code 消费同一错误码表。
- JSON-RPC 承载：`error.code` 用统一应用值（如 `-32000`），规范保留码 `-32700`/`-32601`/`-32602` 用于协议层错误（解析失败/方法不存在/参数不合法）；PackGradle 错误真身放 `error.data`。
- 动作可用性 reason code 仍只出现在能力投影，不作为调用错误抛出（旧栈口径延续）。

**类型链**

- 分层固定：Domain Model → Application → Protocol Contract/DTO；往下才分叉 JSON-RPC 与 TypeScript 生成。TS 类型只从协议契约层生成，不直接从域模型生成。
- 生成产物提交进仓（对标旧栈 wails3 bindings 惯例），Desktop 以本地包消费（承 ADR-0017）。
- 回调参数方法形态废止（旧栈 `AttachQuickUpdateResult(notify func(...))` 之弊）；需要回调的场景一律表达为事件通知。

**命名**

- 方法名「域/动词」样式（`workspace/open`、`workspace/close`、`workspace/list`）；协议内部方法占 `$/` 前缀（`initialize`、`$/cancelRequest`）。完整方法清单归 #123 与实现票。

## 1. 边界与遗留

- 不进本 ADR：`initialize` 字段级 JSON 草案、TS 生成器实现（注解处理或反射扫描）、协议层在 Gradle 多项目的模块落位（随第一张脚手架票，见 ADR-0017）——毕业为实现票。
- 客户端侧消费形态（React 受控重查管线、能力/可用性投影字段口径）归 #123；事件表与序号分配的存储落位归 #121（本 ADR 已清零其跨重启持久化约束）；工作区创建/删除/注册归 #120/#121。
- force-close 未定义，真需要时再议。
- 跨客户端写入的变更，本端 Core 与客户端如何感知与刷新（事件只覆盖本 Core 实例，无跨进程推送）——留父图 fog，随 #122/#121 重审。
