# Agent Native UI Analysis

本仓库用于整理智能体原生 UI 的参考分析，以及 Miracle_Astock 的前台产品—工程一体化目标设计。

## 当前设计主入口

**[Miracle_Astock · 前台产品—工程一体化总设计](Miracle_Astock_FRONTEND_MASTER_DESIGN.md)**

这是当前讨论的唯一主文档。覆盖产品全貌、一级入口、固定底栏、页面与任务生命周期分离、全局“指给 AI 看”、正式/临时对话、代码职责、状态与配置、验收要求。后续讨论修改同一份正文，以 Git 保存修订历史，不新增互相竞争的 V2/V3 文件。

文中区分“已确认”“建议”“待核验”。设计确认不代表代码已经实现或验收通过。当前仅维护设计文档，不授权修改 Miracle_Astock 业务代码或运行资源；目标设计明确后再按其改造现有实现，不以兼容旧组织方式为约束。

## 参考分析

- [agent-native-ui-design.md](agent-native-ui-design.md)：从产品、交互和 UI 设计角度总结 Pen 与同花顺 Lumi 的共同模式。
- [agent-native-engineering-audit.md](agent-native-engineering-audit.md)：从对象模型、运行时上下文、能力接口、状态归属、配置边界、代码结构和验收行为角度做工程落地审计。

分析对象：Pen / pen.dev 的结构化设计文档编辑器，以及同花顺远航版 Lumi 的业务客户端智能体操作层。两份分析中的通用目录、布局和接口是参考，不自动成为 Miracle_Astock 的最终合同。

说明：Lumi 的外部宿主管道没有在本仓库中做真实端到端联调，相关结论以静态源码、技能配置、路由规则和验收案例为依据。
