# PackGradle-Core

PackGradle 的 Java Core（pg-core）重建工程：模组包项目源（Packwiz）与运行实例（Prism）之间受管同步的核心进程与 CLI。每个客户端 spawn 自己的 Core 子进程（ADR-0016）；单一产物两种运行模式——`packgradle` 默认 CLI，`packgradle core` 为子进程模式（ADR-0017）。

- 母项目与决策图：[skyraah/PackGradle](https://github.com/skyraah/PackGradle)（wayfinder 图 #114；Go 参考实现与 ADR 0001–0015 留在母仓）
- 本仓库决策记录：[docs/adr/](docs/adr/)（0016 起）
- 术语表：[CONTEXT.md](CONTEXT.md)；概念地图与图快照：[docs/](docs/)
- 状态：架构决策阶段（图 #114 进行中），尚无生产代码
