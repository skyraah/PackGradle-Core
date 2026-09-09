---
status: accepted
date: 2026-09-10
---

# 0017 · pg-core 技术基线与仓库物理形态

pg-core（Java Core）重建图（wayfinder #114）票 #118 的决策产物（grilling 2026-09-10，两轮收口，均按推荐定档）。输入：研究票 #117 Java 生态事实矩阵、#126 系统保险柜生态、[ADR-0016](0016-pg-core-process-model.md) 进程模型。裁决依据：图定裁决尺（更容易理解 > 更容易调试 > 更容易重构 > 更少运行时魔法 > 更少隐式依赖 > 更少基础设施 > 更少需要记忆的规则），服务单人长期维护。

## 0. 决议清单

- **JDK 25（Temurin 发行版）**；分发用 jpackage 打包自带运行时，用户免装 Java。选 25 不选 21：全新代码库无存量约束；Temurin 免费更新窗口 25 至 2031-09（21 至 2029-12）；25 额外自带 HKDF（凭据信封加密用得上，见 #126——21 需手写 RFC 5869）、AOT cache（JEP 514/515，改善 CLI 冷启动）、jpackage 运行时镜像更小。
- **线程模型**：平台线程为默认；虚拟线程仅在大量并发 IO 处按需使用（并发哈希、批量下载扇出）；文件监听循环固定放专用平台线程。不立「虚拟线程优先」的全局规则。
- **构建链**：Gradle + Kotlin DSL（静态类型、IDE 补全；Groovy DSL 动态类型运行时才报错）。Gradle 多项目结构（core/cli/protocol 等拆分与否）随第一张脚手架票定，本 ADR 只锁构建工具与 DSL。
- **代码分层**：普通 package 分层（按概念地图七域能力分组切包），暂不引入 JPMS。module-info 的强封装收益单人项目吃不到；若将来 jlink 裁剪需要模块化，届时补单个 module-info，归「打包分发」决策。
- **库选型（无框架优先）**：
  - HTTP：JDK `java.net.http.HttpClient`（零依赖；CF API 场景用不到 HTTP/3，无需第三方客户端）。
  - JSON：Jackson 3.1 线（2026 年当前维护线；3.0 为过渡版，官方建议直上 3.1）。
  - SQLite：xerial sqlite-jdbc + `PRAGMA user_version` 自写迁移器（读版本 → 按序执行迁移、每步一个事务 → 写新版本号）。Flyway 的 SQLite 支持已拆去社区仓库，不引入；手写版本号迁移是 SQLite 惯用法，逻辑透明。
  - CLI：picocli 4.7.x。
  - 日志：SLF4J + logback-classic。
- **产物形态（承接 ADR-0016）**：单一构建产物、两种运行模式——`packgradle` 默认进 CLI；`packgradle core` 为 Core 子进程模式（各客户端 spawn 的目标）。CLI/GUI spawn 同一产物的 core 模式；不分发两个可执行，杜绝版本错配。
- **仓库物理形态**：独立仓库 `skyraah/PackGradle-Core`（本仓库）。pg-core 全套（Java 源码、Gradle 构建、术语表、概念地图、决策记录 0016 起）落本仓库；母仓 PackGradle 保留 Go 参考实现、frontend 与 ADR 0001–0015（Go 时代决策，尸体在母仓，`docs/reference-index.md` 指路）。TS 客户端生成链放本仓库（从 Java 协议定义生成 TS，Desktop 以本地包引用消费，具体形态归 #119/#123）；协议变更时两仓协作（两个 PR）。建仓与种子迁移即本 ADR 的执行动作。

## 1. 备注：Gson 与 Jackson 并存的可能

若 #119 协议选型定 lsp4j 的 jsonrpc 模块，其传输层依赖 Gson——届时 Gson（协议传输）与 Jackson 3.1（域对象/DTO 序列化）并存。不构成阻塞，#119 审时把这条成本摆上桌再议。

## 2. 边界与遗留

- 打包分发的余项未决，留父图 fog：安装包格式（msi/exe/dmg 等）、是否启用 AOT cache、版本更新通道。打包方式已定 jpackage 自带运行时（native-image 不取）。
- 术语表与概念地图随本仓库演化（`CONTEXT.md`、`docs/concept-map.md`），母仓根 `CONTEXT.md` 冻结为 Go 时代母本。
