# pg-core 决议快照（map-state）

决策图：PackGradle 母仓 wayfinder 图 [skyraah/PackGradle#114](https://github.com/skyraah/PackGradle/issues/114)（权威记录始终在该 GitHub issue 上）。本文件随每张票收口刷新，是图进度在本仓库的快照。Go 参考实现、旧 ADR 0001–0015 与决策图 issue 都在母仓，`docs/reference-index.md` 指路。

## Destination

pg-core（Java Core）重建的架构决策集：所有阻塞实现的决策各有 ADR/决议 + 模块边界图 + 迁移与验证路线，达到「可以开第一张实现票」的程度。图只出决策：prototype 票的产物是决策证据，不是生产代码。

## Notes（裁决环境）

- **Demo 锚点**：CLI 最小闭环（创建工作区→扫描→差异→计划→Apply→查历史→回滚），Core 独立进程运行；无 watcher、无凭据、无合并、无下载物化。
- **Java 公理（不重审）**：「java 对于我比较熟悉，且更方便抽象化/模块化，保持低耦合」——用户原话。
- **裁决尺（两案接近时按序优先）**：更容易理解 > 更容易调试 > 更容易重构 > 更少运行时魔法 > 更少隐式依赖 > 更少基础设施 > 更少需要记忆的规则。服务单人长期维护，防早期过度复杂化。
- **旧 ADR 处置**：0001–0015 全部重开；方式=强参考默认——Go ADR 决策列为现状候选，维持不需论证，推翻/简化需说清理由。pg-core 新决策落本仓库 docs/adr/ 续编号（0016 起）。
- **Go 栈状态**：立即停更，含 bug fix（2026-09-10 定）；数据迁移不做（开发期无存量数据）。Go 实现只作业务行为与边界情况参考，不机械翻译 package。
- **客户端深度**：CLI = v1 一等公民；VS Code = 只定接入契约；Desktop = React UI 照 ui/react 分支节奏自迁（用户自写），换芯（bindings→protocol 客户端）为图内后置票。
- **工作节奏**：非心脏票 = agent 备料（research/对比矩阵/草案）→ 用户收口 grilling；事实面子代理查，价值判断归用户。

## Decisions so far

- [#117 [research] Java 生态事实矩阵（JDK/库/打包/LSP）](https://github.com/skyraah/PackGradle/issues/117)：JDK 21/25 Premier 至 2028-09/2030-09（LTS 两年节奏）；sqlite-jdbc 活跃、Flyway-SQLite 已社区化（手写 PRAGMA user_version 为惯用）；Jackson 2.21/3.x 双线、picocli 4.7.7、lsp4j 1.0.0（jsonrpc 模块可独立使用）；JDK WatchService macOS 为轮询实现；jpackage + JEP 514/515 AOT cache。全文见票内评论与母仓 research/java-eco-matrix 分支。
- [#126 [research] Java 系统保险柜生态（keyring/DPAPI）](https://github.com/skyraah/PackGradle/issues/126)：java-keyring 停更约 3 年且 Windows 后端 CredWriteA 逼近凭据管理器 2560 字节上限；JNA 自带 DPAPI 但无 Wincred/Keychain 绑定；macOS SecKeychain 被弃用（TN3137）且有 Gradle daemon 长跑失效 issue；纯 JDK 信封零新依赖可行（AES-GCM+PBKDF2 标准自带，HKDF 在 JDK 25 定稿、21 需手写 RFC 5869）。全文见票内评论与母仓 research/java-keyring-eco 分支。
- [pg-core 领域模型与能力域重审（概念地图 + 分组）](https://github.com/skyraah/PackGradle/issues/115)（2026-09-10）：十核心概念全保留；时代词三删（legacy 识别/切换/退场，Java 对旧关联不识别当普通内容）；补五词条（扫描/扫描快照/差异/计划/同步范围，受管范围→同步范围更名）；契约面四词归 #119/#123；七域能力分组定稿（关系与端点 7/差异与决议 10/执行与恢复 8/历史与对象库 7/下载与凭据 4/监听 2/诊断 2）；概念地图落本仓库 `docs/concept-map.md`（票 #118 建仓时自母仓 pg-core/ 迁入）。
- [pg-core 进程模型与生命周期（daemon/单实例/锁归属）](https://github.com/skyraah/PackGradle/issues/116)（2026-09-10）：无共享按需进程——每客户端 spawn 自己的 Core 子进程（CLI 每命令一个、完即退；GUI 客户端打开期间各持一个），不设 daemon、无跨客户端共享、无服务发现；单实例收窄到同类型 GUI 客户端（配置管理须读时加载+原子写）；存储协调沿用 Go（WAL + busy_timeout + active-task 数据库检查，不加跨进程文件锁，数据库是唯一跨进程协调点）；断连即终止、在途任务靠下次启动恢复，无宽限窗口。决议全文见票内评论与本仓库 `docs/adr/0016-pg-core-process-model.md`。
- [pg-core Java 技术基线与仓库物理形态](https://github.com/skyraah/PackGradle/issues/118)（2026-09-10）：JDK 25（Temurin）+ jpackage 自带运行时分发；平台线程默认、虚拟线程仅大量并发 IO 点状使用；Gradle + Kotlin DSL；普通 package 分层（七域切包）不上 JPMS；库=JDK HttpClient / Jackson 3.1 / xerial sqlite-jdbc + PRAGMA user_version 自写迁移器 / picocli / SLF4J+logback-classic；单一产物两种模式（`packgradle` 默认 CLI、`packgradle core` 子进程）；独立仓库 PackGradle-Core（本仓库）承接 pg-core 全套与 TS 客户端生成链，旧 ADR 0001–0015 留母仓，协议变更两仓协作。决议全文见票内评论与本仓库 `docs/adr/0017-pg-core-tech-baseline.md`（含 Gson/Jackson 并存备注，归 #119 计成本）。
- [pg-core Protocol 与事件流选型](https://github.com/skyraah/PackGradle/issues/119)（2026-09-10）：JSON-RPC 2.0 over stdio（LSP 式 Content-Length 长度帧；自写薄层七件不引 lsp4j，Gson 并存备注作废）；**单 Core 多工作区前提**、Workspace 一等协议实体（显式 `workspace_id` 路由 + `workspace/open`/`close`/`list` 生命周期，重复调用幂等，close 遇运行中 task 返 `err.workspace.busy`）；`initialize` 严格相等握手（`protocol_version` 整数首版 1、`capabilities`=方法+事件清单；查询面 DTO 的 `schema_version` 全砍）；长操作一律立即返 `task_id` + `task_updated` 事件通知；单一 live 通知流（信封 `{schema_version, stream_epoch, stream_sequence, event_id, event_type, workspace_id, emitted_at, relation_id, task_id, payload}`，epoch 变化即重查，无重放无订阅状态，**#121 序号持久化约束清零**）；`err.<域>.<原因>` 字符串码延续（JSON-RPC 统一应用 code、真身在 `error.data`）；TS 类型只从 Protocol Contract/DTO 层生成（Domain→Application→Protocol Contract 分层）。决议全文见票内评论与本仓库 `docs/adr/0018-pg-core-protocol.md`。

## 票面快照（2026-09-10 · 票 #119 收口时；#115/#116/#117/#118/#126 同日先已收口）

| 票 | 标题 | 状态 |
|---|---|---|
| #115 | 领域模型与能力域重审 | **已关** |
| #116 | 进程模型与生命周期 | **已关** |
| #117 | [research] Java 生态事实矩阵 | **已关** |
| #118 | Java 技术基线与仓库物理形态 | **已关** |
| #119 | Protocol 与事件流选型 | **已关**（本票，ADR-0018） |
| #120 | 领域语义重审·能力域组 | 边界可取（按七域分组拆票） |
| #121 | 存储与持久化边界 | 边界可取 |
| #122 | 任务编排与 watcher 架构 | 被阻塞（1：#120；#119 已解） |
| #123 | 三客户端接入契约 | 边界可取（#119 已解阻塞） |
| #124 | 迁移与行为对齐策略 | 被阻塞（1：#121） |
| #125 | Desktop host 选型（低优先） | 被阻塞（1：#123） |
| #126 | [research] Java 系统保险柜生态 | **已关** |
| #127 | 凭据机制重估 | 被阻塞（1：#120） |

## Not yet specified（fog）

- 下载物化与 CF 通道的 Java 形态（免钥匙直链重估；旧 ADR-0005/0008）
- 诊断/日志/脱敏横切（承接已关 #113 的能力面 + ADR-0011 会话日志/别名路径重估）
- 打包分发余项（安装包格式、AOT cache 启用与否、版本更新通道——打包方式已定 jpackage 自带运行时，见 #118/ADR-0017）
- i18n/本地化归属（Core vs 客户端）
- 性能基线（大包扫描、CAS 冷链路——承接已关 #69 的线索）
- 多客户端并发打开同一工作区（进程级口径已定于 #116：允许并存、数据库是唯一协调点；协议侧已定于 #119：事件只覆盖本 Core 实例、无订阅状态、恢复一律重查；跨客户端写入的可见性与刷新触发待 #122/#121）
- 测试与 CI 形态（含 TS 客户端链）

## Out of scope

- 生产代码实现（图只出决策；spike 产物为证据）
- VS Code 扩展实现（只定契约，实现等 Core 稳定）
- 数据迁移/双栈兼容（无存量数据；Go 已停更）
- Desktop 前端 UI 迁移本身（照 ui/react 既有节奏走，不属本图）
- 语言重审（Java 为公理）
