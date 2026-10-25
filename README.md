# Agent Native UI Analysis

这份仓库收录两份关于智能体原生 UI 的总结文档：

- `agent-native-ui-design.md`：从产品、交互和 UI 设计角度总结 Pen 与同花顺 Lumi 的共同模式。
- `agent-native-engineering-audit.md`：从对象模型、运行时上下文、能力接口、状态归属、配置边界、代码结构和验收行为角度做工程落地审计。

分析对象：

- Pen / pen.dev：结构化设计文档编辑器。
- 同花顺远航版 Lumi：业务客户端内的智能体操作层。

说明：Lumi 的外部宿主管道没有在本仓库中做真实端到端联调，相关结论以静态源码、技能配置、路由规则和验收案例为依据。
