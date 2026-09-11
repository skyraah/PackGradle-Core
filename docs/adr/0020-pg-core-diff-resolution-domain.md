---
status: accepted
date: 2026-09-11
---

# 0020 · pg-core 差异与决议域语义

pg-core（Java Core）重建图（wayfinder #114）票 #129 的决策产物（grilling 2026-09-11 两轮收口：R1 八问 + R2 两问，全部按推荐通过；用户对 Q3 补强作用域、stale 查询面语义与呈现归属）。输入：母仓 ADR-0005 §5（.index 只读）、ADR-0009（合并语义，强参考默认）、ADR-0010 §8（扫描成本模型）、ADR-0013（决议忽略与手动处理）、[ADR-0019](0019-pg-core-relation-endpoint-domain.md)（基线/计划/修订号/历史按 Relation 隔离，本域直接继承）、`docs/concept-map.md` 域 2、Go 实现事实（票 #129 备料评论：分类 11 种、决议 8 种、计划操作 6 种、确认码实出 4、失效三机制、hash cache 五元组键、direction=ignore 生效点七处、ADR-0009 合并已在 P4 票 #87/#93/#94 全量实现）。裁决依据：图定裁决尺（更容易理解 > 更容易调试 > 更容易重构 > 更少运行时魔法 > 更少隐式依赖 > 更少基础设施 > 更少需要记忆的规则），服务单人长期维护。

## 0. 决议清单

**一、域语义维持大盘——全套维持（Q1）**

Go 已验收语义原样定为 pg-core 语义，逐项：

- **差异分类 11 种**：`noop`、`project_to_runtime`、`runtime_to_project`、`remove_runtime_candidate`、`remove_project_candidate`、`converged`（双端同变同指纹，含双端同删）、`conflict_modify`、`merged_clean`、`conflict_delete_modify`、`adopt_equal`（无基线双端同语义）、`init_choice`。三方比较以基线为公共祖先、两端最新扫描快照为输入，资源集合取三方并集；低置信度 mod 绝不参与跨侧等价；双侧同改 digest 不同先查合并判定，合并面不可用维持 `conflict_modify`。
- **决议 8 种与矩阵**：`adopt_equal` / `initialize_from_project` / `initialize_from_runtime` / `take_project` / `take_runtime` / `skip` / `manual` / `take_merged`。`init_choice` 接受「initialize_from_project / initialize_from_runtime / skip / manual」；`modify_modify` 与 `delete_modify` 接受「take_project / take_runtime / skip / manual」；其余组合一律拒绝。「从不存在的一侧初始化」拒绝（源侧表示为空 → `err.plan.resolution_invalid`）。
- **计划失效三机制**：修订号递增（仅策略修改，ADR-0002/0019 口径）、绑定指纹失配（重绑）、TTL 15 分钟过期；expired/stale 均为读取时投影，不落库。
- **确认要求实出四码**：`overwrite`（info 级）/ `delete` / `write_project` / `unrecoverable`（源侧低置信度 mod 的操作）。
- **同步范围规则形状与编译约束**：恰好一条 mod 规则（两侧前缀归一化后为 mods）、文件规则前缀必填 root-relative 且禁入 mods/、include/exclude glob 编译期证明、最长前缀胜出、并列剔除走 `diag.mapping.collision` 诊断、无规则命中回退 bidirectional。direction=ignore 生效点七处维持——计划构建剔除、操作级方向门、方向编解码进 Conflict.Detail，以及差异面四处（verifyRescan、diff_state、changes 页、QuickUpdate no_diff）；**扫描面不生效**：被忽略资源照常进扫描快照，只在差异调用方与计划层过滤。
- **低置信度 mod 身份纪律**：modrinth/curseforge 编号高置信度、仅凭路径低置信度（`mod:jar:` 等）；运行实例侧经扫描 hint 复用项目侧 ResourceID，是唯一跨侧身份通道。
- **.index 只读**：结构性保证「不观察即不计划」——项目侧只观察 index.toml 列出的 metafile、运行实例侧跳过 `.index` 目录条目（内容只作只读元数据源），不设显式路径黑名单；合并黑名单与文件规则禁入 mods/ 连带覆盖。
- **上游变更**：纯叙事、无独立检测机制；察觉通道只有扫描（手动、快速更新链内、watcher 自动链）。「外部写者」只作竞态防御用语（取数复核），不是领域概念。
- **快照与事件**：「每端最新一份」参与比较；头表四指纹（binding_fingerprint / normalization_version / policy_digest / snapshot_digest）记录链维持；`relation_invalidated` 发射点维持（扫描成功终态、Apply/restore committed、QuickUpdate 停靠 awaiting_confirmation 补发、恢复收口、重绑事务提交后）。

**二、扫描成本模型——维持（Q2）**

全枚举 + hash cache，增量枚举协议不做（ADR-0010 §8 继承）。缓存键五元组（RootFingerprint + 小写斜杠相对路径 + size + mtime(UnixNano) + FileKey）维持，**「键含文件身份通道」的正确性要求维持**——保 mtime 的替换文件不得假命中旧哈希。缓存定位不变：性能优化、可随时丢弃、不是事实来源。Windows 下 FileKey 实现通道（JDK 标准库无直接等价物，`BasicFileAttributes.fileKey()` 在 Windows 返回 null；JNA `GetFileInformationByHandle` 或退化方案）留实现期选型，不在本 ADR 裁。

**三、快照策略失配门——新增决议（Q3/Q9/Q10；Go 无门，唯一新增机制）**

- 凡以扫描快照为输入的计划生成入口——同步计划（PrepareSync）与回滚计划——统一校验快照 `policy_digest` 与当前策略摘要一致；不一致拒绝生成、返回 `err.sync.snapshot_stale`；不为同步与回滚建两套例外规则。
- 门表达的是「该快照不得作为新计划的输入」，不表示快照本身不可读取。已存在的 stale 计划仍可查询、列出、清理；不要求所有计划查询接口统一报错。
- 只读差异视图（changes 页、diff_state 等）继续允许读取失配快照，不报错不拒绝；动作面（创建同步/回滚计划）严格拒绝。
- 是否在失配时自动触发重扫与排队归 #122（编排）；是否在 UI/协议结果加「快照早于当前策略」标记归 #123（契约呈现），本票不新增呈现字段。
- 动机：原边界情况——策略新增前缀但未重扫 → 旧快照建计划 → 一路执行到收尾复扫才发现未处理资源 → `unselected` violation 使整场 `verify_mismatch` 失败。安全但报错离根因远；门把失败提前到语义正确位置（裁决尺「更容易理解 > 更容易调试」），代价只是一条比较逻辑与一个错误码。

**四、计划生命周期——维持（Q4）**

draft / resolved 无限并存、无主动清理；既有两个消费点维持（重绑预检计数 `invalidated_plan_count`、「最新一张待人工」投影排除已被 resolved_from 推进的祖先 draft）；过期/作废读取时计算不写库；删除工作区时计划随关系清理（ADR-0019）。不加保留策略（堆积实测再说）。

**五、mod 持久排除——维持不做（Q5；ADR-0013 §4 遗留「另票」的裁决落点）**

mods 恒由唯一 mod 规则管理；mod 冲突的 ignore 只对本次决议生效（不合成同步范围规则）；不新增 mod 级忽略规则形状。高频需求真实出现再开票。

**六、RuntimeLocalPolicy——删除（Q6；ADR-0013 Q8-a 遗留落点）**

Java 重建不带该字段，exclude / report 两枚举不保留（Go 侧该字段无任何运行时行为分支，属死字段）。现状行为写进语义：运行实例本地 mod（hint 未命中）恒为低置信度观察 + `diag.scan.runtime_local` 诊断，不参与跨侧等价。

**七、死建模清债——不带（Q7）**

`ConflictKind.identity_ambiguous` 与 `mapping_collision`（无生产者；规则前缀并列实际走 `diag.mapping.collision` 诊断，不进冲突表）、确认码 `shared_materialization`（只在注释出现、从不产出）、`LogicalResource` 类型（注释称供合并用、实际闲置）、`PlanConfirmed` 状态（schema CHECK 有、代码从不写入）——Java 模型一律不建，行为不变。「不机械翻译 Go」的正向应用；无 ADR 决议被推翻（任何 ADR 都未决议过要它们）。

**八、合并域——ADR-0009 全套维持定稿（Q8；Demo-out，语义照常定义）**

- **事实修正**：ADR-0009 并非「纯决策零实现」——Go 已在 P4 票 #87/#93/#94 全量实现（`epiclabs-io/diff3` pin commit 接入、`merged_clean`/`take_merged`/`write_merged` 分类-决议-操作三件套、暂存期确定性重算、合并预览、冲突块 detail JSON），有测试与验收背书。
- 全套语义维持定稿：文本级 diff3 三方合并；未冲突区域字节级不变；冲突块=每资源一行 + detail JSON（无损存储、有损查询）；`merged_clean` 属非冲突操作（授权模式免确认口径延续），含冲突块永不进自动面；二进制与 `.index` 永不合并；合并结果按资源类型校验、失败降级 `conflict_modify`（块证据保留）；计划期出证据、暂存期按计划锁定内容确定性重算；执行期前置条件=双端字节与快照相符。
- 「合并产物一律入对象库保全」作为领域语义保留；对象库 schema、磁盘布局、引用与 GC 归 #131 / #121，本 ADR 不展开。
- Java 三方合并库选型不立 research 票：写入图 Not yet specified，待合并实现排期前立票。

## 1. 边界与遗留

- 归属移交：verifyRescan / `unselected` violation 与复扫校验归 #130；扫描触发、创建段互斥、自动链编排、失配自动重扫归 #122；孤儿快照与计划引用的 GC、对象库处置归 #131；download 操作行字段与物化归 #132；失配 UI 标记与契约面 DTO/方法形态归 #123。
- `err.*` 字符串码延续（ADR-0018 口径）；新增 `err.sync.snapshot_stale` 入错误码族。
- 概念地图无需增删概念（域 2 仍 10 个）；合并词条备注更新为「ADR-0009 全套维持定稿（ADR-0020）」；CONTEXT.md 合并/冲突块词条补 ADR-0020 指针。
- Go 侧遗留事实存档（Java 重建参考）：规则级 materialization 合法枚举仅 `copy`，download 只是计划操作行字段、策略面不可声明（归 #132 重估的输入）；冻结旧栈 `service.PushMeta/PullMeta` 仍写 `mods/.index`（新旧栈对 .index 只读不一致，旧栈冻结不修）。
