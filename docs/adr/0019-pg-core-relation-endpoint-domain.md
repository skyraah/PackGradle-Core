---
status: accepted
date: 2026-09-11
---

# 0019 · pg-core 关系与端点域语义

pg-core（Java Core）重建图（wayfinder #114）票 #128 的决策产物（grilling 2026-09-10/11，两轮收口：R1 六问 + R2 端点清理，全部按推荐通过，用户补强删除语义与端点不变量）。输入：母仓 ADR-0002（修订号）/ ADR-0003（预检两段式与单事务，强参考默认口径）、[ADR-0016](0016-pg-core-process-model.md) 进程模型、[ADR-0018](0018-pg-core-protocol.md) 协议（Workspace 一等实体）、`docs/concept-map.md` 域 1、Go 实现事实（票 #128 备料评论：修订号唯一递增源是 SavePolicy、四类乐观锁消费点、RunInTx 单事务落地、端点行跨关系幂等复用、Go 全代码无删除操作）。裁决依据：图定裁决尺（更容易理解 > 更容易调试 > 更容易重构 > 更少运行时魔法 > 更少隐式依赖 > 更少基础设施 > 更少需要记忆的规则），服务单人长期维护。

## 0. 决议清单

**修订号（乐观锁）——全套维持，防护对象重述**

- 防护对象重述：从「daemon 内并发防护」改为「多个客户端进程（CLI / Desktop / 其他）通过 SQLite 共享数据库时的并发写入校验」。daemon 已移除（ADR-0016），但跨进程并发写入仍是合法场景，且 SQLite 是唯一跨进程协调点——数据库内的乐观锁即权威校验，不随 daemon 一并退役。
- 递增源唯一维持：仅策略修改（SavePolicy）递增；创建即第 1 代、初始 policy 写入不算修改（ADR-0002 决议 1/2 原样）。
- 消费点四类维持：策略编辑 `err.mapping.stale_revision`、PrepareSync 入口 `err.sync.revision_mismatch`、计划确认 `err.plan.stale`、回滚读投影。
- ADR-0002 其余决议无新压力、原样维持：UI 不展示数字（内部一致性字段）、修订号与策略集版本双计数独立、契约面保留 revision 回传字段。

**工作区实体形态——1:1 投影维持**

- Workspace ↔ Relation = 1:1。Workspace 是 Relation 在产品/协议层的用户可见投影与身份；协议 `workspace_id`（ADR-0018）即关系标识。
- 领域核心使用 Relation 作为实体语义；不引入「一 Workspace 多 Relation」聚合结构。

**端点共享——合法形态**

- 项目源 / 运行实例可被多条关系引用；端点身份独立于 Relation 状态。
- baseline / plan / revision / history 均按 Relation 隔离——#129 差异域按「基线每关系独立」口径直接继承。

**预检两段式与单事务 doctrine——全套维持（ADR-0003）**

- Prepare 与 Apply 各自是独立、合法的领域操作；预检是持久化实体（限时有效、仅可消费一次、`prep_consumed` / `prep_expired` 两码拆分维持）。
- CLI 单命令创建工作区 = 客户端串联 Prepare → Apply，不提供绕过预检持久化检查的旁路。
- 单事务 doctrine 维持：多步元数据写入收进一个 SQLite 事务；「先提交事务，再发布事件」全局规则维持。

**重绑——定稿（Demo-out，语义照常定义）**

- 替换单侧端点根路径；Relation identity 不变；原位更新端点引用（adapter 不变）。
- Apply 后清除旧 baseline、必须重新初始化；不提供 baseline inheritance；旧版「等价证明留 Phase 2」待办语义删除——恒重走初始化即终态语义。
- revision 不递增；旧 plan 因 binding fingerprint mismatch 失效。

**删除——新增 Relation 生命周期语义（Go 从未实现）**

- 删除是 Relation 生命周期的一等操作；产品层名称 Delete Workspace（删除工作区）。
- 前置条件：无活跃任务、不处于恢复所需；前置检查与删除执行同事务（数据库是唯一跨进程协调点，检查必须与写入同事务才权威）。
- 删除范围：Relation 行及其关系级依附状态（计划 / 任务 / 预检）；History / Commit / CAS 对象的后续处置显式移交 #131，本 ADR 不提前决定其保留或 GC 语义。
- 终态操作：不引入 soft-delete / undelete。

**端点清理——删除语义的一部分，不是对象库 GC**

- Endpoint 是当前关系的共享实体，不是历史记录。删除 Relation 时：仍被其他 Relation 引用的 Endpoint 必须保留；删除后已无任何 Relation 引用的 Endpoint 在同一事务内清理。
- Endpoint 不进入「孤儿」持久状态；不将该清理委托给 #131 的 CAS GC。
- 数据库不变量：**任意存活 Endpoint 至少被一条存活 Relation 引用。**

## 1. 边界与遗留

- 维持项不另行成文（强参考默认、无新压力）：策略集版本双计数独立（ADR-0002 决议 5）、Relation health 四态（healthy / endpoint_missing / rebind_required / recovery_required）、预检限时有效与消费守卫、端点占用检查（UNIQUE + 指纹复核）与创建幂等复用。
- 契约面 revision 字段形态归 #123；DTO 与方法签名归协议契约层实现票。
- History / Commit / CAS 在删除后的处置归 #131（其票面已补移交说明）。
- 概念地图随本票更新：域 1 增「删除工作区」概念（7 → 8），修订号挂起项清账；CONTEXT.md 词条同步（修订号重述、重绑删旧尾巴、新增删除工作区）。
